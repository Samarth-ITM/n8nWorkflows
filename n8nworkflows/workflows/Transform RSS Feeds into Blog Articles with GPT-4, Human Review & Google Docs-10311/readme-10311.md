Transform RSS Feeds into Blog Articles with GPT-4, Human Review & Google Docs

https://n8nworkflows.xyz/workflows/transform-rss-feeds-into-blog-articles-with-gpt-4--human-review---google-docs-10311


# Transform RSS Feeds into Blog Articles with GPT-4, Human Review & Google Docs

### 1. Workflow Overview

This workflow automates the transformation of RSS feed articles into polished blog posts using GPT-4 AI, incorporates human review via gotoHuman, and publishes approved articles to Google Docs while notifying the team on Slack. It is designed for content teams, bloggers, marketing agencies, and news sites aiming to streamline content creation with quality control.

**Logical Blocks:**

- **1.1 Scheduled Trigger & RSS Fetching:** Automatically triggers every 6 hours to read new articles from a specified RSS feed.
- **1.2 Data Extraction & Formatting:** Processes RSS data to extract key information such as title, keywords, and description, preparing it for AI generation.
- **1.3 AI Article Generation:** Uses GPT-4 (via LangChain agent) to produce a detailed, structured blog article based on extracted RSS content.
- **1.4 Article Structuring:** Parses and organizes AI output into components like title, body, metadata, and word count.
- **1.5 Human Review:** Sends the draft article to a reviewer through gotoHuman and awaits approval or rejection.
- **1.6 Approval Routing:** Routes the workflow based on approval status to either publish or reject the article.
- **1.7 Publishing:** Creates a Google Doc with the approved article content and metadata.
- **1.8 Team Notification:** Posts a formatted notification message to a Slack channel about the published article.

---

### 2. Block-by-Block Analysis

#### Block 1.1: Scheduled Trigger & RSS Fetching

**Overview:**  
Triggers the workflow every 6 hours and fetches the latest articles from the configured RSS feed.

**Nodes Involved:**  
- Schedule Trigger  
- RSS Read  

**Node Details:**

- **Schedule Trigger**  
  - Type: Schedule Trigger  
  - Role: Initiates the workflow on a fixed time interval (every 6 hours).  
  - Configuration: Interval set to 6 hours.  
  - Input: None (trigger node).  
  - Output: Starts RSS Read node.  
  - Edge Cases: If the system clock changes or n8n is down, scheduled runs may be delayed or missed.

- **RSS Read**  
  - Type: RSS Feed Read  
  - Role: Reads new articles from the RSS feed URL.  
  - Configuration: RSS URL set to `"https://example.com/rss-feed.xml"` (user should replace with their feed).  
  - Input: Trigger from Schedule Trigger.  
  - Output: Passes RSS feed items to the next node.  
  - Edge Cases: RSS feed could be down, malformed, or empty; network errors possible.

---

#### Block 1.2: Data Extraction & Formatting

**Overview:**  
Extracts and formats the necessary fields from the RSS article data to prepare for AI article generation.

**Nodes Involved:**  
- Format RSS Data (Code node)  

**Node Details:**

- **Format RSS Data**  
  - Type: Code (JavaScript)  
  - Role: Parses RSS item fields such as title, description, keywords, and publication date.  
  - Configuration:  
    - Processes the first RSS item only (can be extended for multiple articles).  
    - Extracts topic/title, description, keywords (simple heuristic extracting words >2 chars), and other metadata.  
    - Sets default keywords if none found.  
    - Defines article target length and tone.  
  - Input: RSS Read node output.  
  - Output: JSON with formatted article data for AI consumption.  
  - Edge Cases: No RSS items (throws error), missing fields handled with defaults, keyword extraction may be naive.

---

#### Block 1.3: AI Article Generation

**Overview:**  
Generates a detailed blog article using GPT-4 based on the formatted RSS data.

**Nodes Involved:**  
- Generate Article with AI (LangChain Agent)  
- OpenAI Chat Model (GPT-4)  

**Node Details:**

- **Generate Article with AI**  
  - Type: LangChain Agent  
  - Role: Sends a prompt with RSS data to GPT-4 specifying article structure and style.  
  - Configuration:  
    - Prompt includes article title, summary, source URL, keywords, tone, and detailed structure (introduction, analysis, summary).  
    - Uses system message to set AI’s role as professional content writer.  
  - Input: Formatted RSS data from previous node.  
  - Output: Raw AI-generated article text.  
  - Edge Cases: API errors, rate limits, prompt malformed, incomplete AI response.

- **OpenAI Chat Model**  
  - Type: LangChain OpenAI Chat Model  
  - Role: Provides backend GPT-4 model support for LangChain agent.  
  - Configuration: Uses GPT-4 model variant `"gpt-4o"`.  
  - Input: From LangChain Agent node.  
  - Output: AI-generated article text.  
  - Edge Cases: API key issues, model unavailability, latency.

---

#### Block 1.4: Article Structuring

**Overview:**  
Processes AI output to extract a clean title, article body, meta description, and calculates word count.

**Nodes Involved:**  
- Structure Article Data (Code node)  

**Node Details:**

- **Structure Article Data**  
  - Type: Code (JavaScript)  
  - Role: Parses the AI’s markdown output, extracting the first-level header as title, separates meta description from article body, and calculates word count.  
  - Configuration:  
    - Uses regex to extract title from lines starting with `# `  
    - Extracts meta description from text following `---` separator.  
    - Removes markdown headers and whitespace to count words.  
  - Input: AI-generated article text.  
  - Output: Structured JSON with article components and metadata for review.  
  - Edge Cases: AI output missing expected format; fallback to original RSS title.

---

#### Block 1.5: Human Review

**Overview:**  
Sends the structured article to a human reviewer via gotoHuman and waits for approval or rejection.

**Nodes Involved:**  
- Request Human Review (gotoHuman node)  

**Node Details:**

- **Request Human Review**  
  - Type: gotoHuman node  
  - Role: Posts article data to a human review interface, allowing a reviewer to approve or reject the article.  
  - Configuration:  
    - Passes fields: topic, keywords, word count, article body, original URL, article title, meta description.  
    - Uses a specific review template ID (must be replaced with user’s gotoHuman template ID).  
    - Waits for user response (approval or rejection).  
  - Input: Structured article data.  
  - Output: Review response JSON with approval status.  
  - Edge Cases: Network issues, invalid template ID, reviewer does not respond, timeouts.

---

#### Block 1.6: Approval Routing

**Overview:**  
Checks the human review response and routes the workflow for publishing or rejection accordingly.

**Nodes Involved:**  
- Check Approval Status (If node)  
- Article Rejected (NoOp node)  
- Create Google Doc (Google Docs node)  

**Node Details:**

- **Check Approval Status**  
  - Type: If node  
  - Role: Checks if the review response equals `"approved"`.  
  - Configuration:  
    - Condition: `$json.response === "approved"`  
  - Input: Review response from gotoHuman.  
  - Output:  
    - True branch: proceeds to Google Doc creation.  
    - False branch: triggers rejection node.  
  - Edge Cases: Missing or malformed response field.

- **Article Rejected**  
  - Type: No Operation (NoOp)  
  - Role: Placeholder node representing rejection handling (e.g., logging or stopping).  
  - Input: False branch from approval check.  
  - Output: None.  
  - Edge Cases: No direct output.

- **Create Google Doc**  
  - Type: Google Docs node  
  - Role: Creates a new Google Document with the article title.  
  - Configuration:  
    - Title set from original article title.  
    - Drive ID and Folder ID left blank (defaults to user’s default Google Drive location).  
  - Input: True branch from approval check.  
  - Output: Document metadata including document ID for further updates.  
  - Edge Cases: Google API auth errors, quota limits.

---

#### Block 1.7: Publishing Article Content

**Overview:**  
Adds the full article content and metadata to the newly created Google Doc.

**Nodes Involved:**  
- Add Article Content (Google Docs node)  

**Node Details:**

- **Add Article Content**  
  - Type: Google Docs node  
  - Role: Updates the created Google Doc by inserting the article body and appended metadata.  
  - Configuration:  
    - Inserts article body text first, then inserts metadata info including source URL, original title, reviewer name, approval date, and word count.  
    - Document URL is dynamically set from the created document ID.  
  - Input: Output from Create Google Doc.  
  - Output: Confirmation of document update.  
  - Edge Cases: Document not found, API errors, permission issues.

---

#### Block 1.8: Team Notification

**Overview:**  
Sends a notification message to a Slack channel with details about the newly published article.

**Nodes Involved:**  
- Send Slack Notification (Slack node)  
- Article Published (NoOp node)  

**Node Details:**

- **Send Slack Notification**  
  - Type: Slack node  
  - Role: Posts a formatted message to a specified Slack channel with article details and review info.  
  - Configuration:  
    - OAuth2 authentication with Slack workspace.  
    - Channel ID must be replaced by user.  
    - Message includes title, word count, keywords, reviewer, approval date, source URL, and meta description.  
  - Input: Output from Add Article Content.  
  - Output: Slack post confirmation.  
  - Edge Cases: Slack API errors, invalid channel, auth failures.

- **Article Published**  
  - Type: No Operation (NoOp)  
  - Role: Terminal node signifying successful article publication completion.  
  - Input: From Slack notification node.  
  - Output: None.

---

### 3. Summary Table

| Node Name            | Node Type                    | Functional Role                              | Input Node(s)           | Output Node(s)           | Sticky Note                                                                                                               |
|----------------------|------------------------------|----------------------------------------------|-------------------------|--------------------------|---------------------------------------------------------------------------------------------------------------------------|
| Sticky Note - Overview| Sticky Note                  | Workflow summary and instructions           |                         |                          | ## 📰 RSS to Blog Article Generator with Human Approval<br>Automatically converts RSS to blog articles with AI and review.|
| Schedule Trigger     | Schedule Trigger             | Triggers workflow every 6 hours              |                         | RSS Read                 | ### 1️⃣ Scheduled Trigger<br>Runs workflow every 6 hours to check for new RSS articles. Adjust interval as needed.         |
| RSS Read             | RSS Feed Read                | Fetches latest RSS feed articles              | Schedule Trigger         | Format RSS Data           | ### 2️⃣ Fetch RSS Feed<br>Reads articles from your RSS feed. Replace URL with your own source.                             |
| Format RSS Data      | Code                        | Extracts and formats RSS article data         | RSS Read                 | Generate Article with AI  | ### 3️⃣ Process Data<br>Extracts title, keywords, and description from RSS feed. Formats for AI processing.                |
| Generate Article with AI | LangChain Agent           | Generates blog article from RSS data          | Format RSS Data          | Structure Article Data    | ### 4️⃣ AI Article Generation<br>Uses OpenAI to generate detailed blog post based on RSS content.                          |
| OpenAI Chat Model    | LangChain OpenAI Chat Model | Provides GPT-4 model for AI generation        | Generate Article with AI |                          | ### 4️⃣ AI Article Generation<br>GPT-4 model backend for article creation.                                                 |
| Structure Article Data | Code                       | Parses AI output into structured article data | Generate Article with AI | Request Human Review      | ### 5️⃣ Format Article Data<br>Structures AI content with title, body, metadata, word count.                               |
| Request Human Review | gotoHuman                   | Sends article for human approval               | Structure Article Data   | Check Approval Status     | ### 6️⃣ Human Review Request<br>Sends article to reviewer via gotoHuman and waits for approval/rejection.                  |
| Check Approval Status| If                          | Routes based on approval status                | Request Human Review     | Create Google Doc, Article Rejected | ### 7️⃣ Approval Check<br>Routes workflow based on review decision. Approved articles proceed to publishing.     |
| Create Google Doc    | Google Docs                 | Creates Google Doc for approved article        | Check Approval Status    | Add Article Content       | ### 8️⃣ Publish to Google Docs<br>Creates new Google Doc with article title and content including metadata.                |
| Add Article Content  | Google Docs                 | Inserts article content and metadata into Doc | Create Google Doc        | Send Slack Notification   | ### 8️⃣ Publish to Google Docs<br>Adds article body and metadata to created document.                                      |
| Send Slack Notification | Slack                     | Sends notification message about published article | Add Article Content      | Article Published         | ### 9️⃣ Team Notification<br>Sends formatted Slack message with article and approval details.                              |
| Article Published    | No Operation (NoOp)         | Marks successful completion of publication     | Send Slack Notification  |                          |                                                                                                                           |
| Article Rejected     | No Operation (NoOp)         | Placeholder for rejected articles              | Check Approval Status    |                          |                                                                                                                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.

2. **Add a Schedule Trigger node**  
   - Type: Schedule Trigger  
   - Configure: Run every 6 hours (set interval to 6 hours).

3. **Add an RSS Feed Read node**  
   - Type: RSS Feed Read  
   - Connect Schedule Trigger → RSS Read.  
   - Configure: Set RSS feed URL to your source (replace `"https://example.com/rss-feed.xml"`).

4. **Add a Code node named "Format RSS Data"**  
   - Connect RSS Read → Format RSS Data.  
   - Paste the JavaScript code that:  
     - Extracts the first RSS item.  
     - Parses title, description, keywords (top 5 words >2 chars).  
     - Sets default keywords if missing.  
     - Adds target length and tone attributes.

5. **Add a LangChain Agent node named "Generate Article with AI"**  
   - Connect Format RSS Data → Generate Article with AI.  
   - Configure:  
     - Prompt includes original title, summary, URL, keywords, tone, target length, and detailed article structure.  
     - Use system message to define AI as professional writer.  
   - Ensure LangChain OpenAI Chat Model node is present and connected as AI language model backend.  

6. **Add a LangChain OpenAI Chat Model node**  
   - Connect to Generate Article with AI as AI languageModel.  
   - Configure model to `"gpt-4o"` (GPT-4).  
   - Set OpenAI API credentials.

7. **Add a Code node named "Structure Article Data"**  
   - Connect Generate Article with AI → Structure Article Data.  
   - Paste JavaScript code that:  
     - Extracts article title from first markdown header.  
     - Separates meta description after `---`.  
     - Extracts article body and computes word count.

8. **Add the gotoHuman node named "Request Human Review"**  
   - Connect Structure Article Data → Request Human Review.  
   - Configure:  
     - Map fields: topic, keywords, word count, article body, original URL, article title, meta description.  
     - Set review template ID (replace `"YOUR_GOTOHUMAN_TEMPLATE_ID"` with your actual template).  
     - Configure gotoHuman credentials.

9. **Add an If node named "Check Approval Status"**  
   - Connect Request Human Review → Check Approval Status.  
   - Configure condition: check if `$json.response === "approved"`.

10. **Add a Google Docs node named "Create Google Doc"**  
    - Connect Check Approval Status True branch → Create Google Doc.  
    - Configure:  
      - Title set to original article title.  
      - Drive and Folder IDs as needed.  
      - Configure Google credentials.

11. **Add a Google Docs node named "Add Article Content"**  
    - Connect Create Google Doc → Add Article Content.  
    - Configure actions to insert:  
      - Article body text.  
      - Metadata block with source URL, original title, reviewer name, approval date, word count.  
    - Use created document ID from previous node as target document.

12. **Add a Slack node named "Send Slack Notification"**  
    - Connect Add Article Content → Send Slack Notification.  
    - Configure:  
      - Set OAuth2 Slack credentials.  
      - Select Slack channel ID (replace `"YOUR_SLACK_CHANNEL_ID"`).  
      - Compose message with article title, word count, keywords, reviewer, approval date, source URL, and meta description.

13. **Add NoOp nodes "Article Published" and "Article Rejected"**  
    - Connect Send Slack Notification → Article Published.  
    - Connect Check Approval Status False branch → Article Rejected.

14. **Test the workflow end-to-end** ensuring:  
    - Valid RSS feed.  
    - OpenAI GPT-4 API access and credentials.  
    - gotoHuman account with configured template and credentials.  
    - Google Docs OAuth2 credentials.  
    - Slack workspace OAuth2 credentials and correct channel ID.

---

### 5. General Notes & Resources

| Note Content                                                                                                                             | Context or Link                                       |
|------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------|
| The workflow requires valid API keys/credentials for OpenAI (GPT-4), Google Docs, gotoHuman, and Slack integrations.                    | Setup Credentials in n8n                             |
| Replace placeholder values such as RSS feed URL, gotoHuman review template ID, and Slack channel ID before production use.               | Configuration instructions within sticky notes       |
| The AI prompt is customizable in the "Generate Article with AI" node to adjust tone, length, and structure of generated articles.        | Located in LangChain Agent node parameters            |
| For multiple article processing, extend "Format RSS Data" and subsequent nodes to handle array processing instead of only first item.    | Code modification recommended                         |
| Slack notifications provide real-time updates to the team upon article publication for better collaboration and monitoring.              | Slack node configuration                              |
| Refer to official n8n documentation for node-specific credential setup and error handling best practices.                                | https://docs.n8n.io/                                  |
| Workflow designed by a senior technical analyst specialized in n8n automation workflows.                                                  | —                                                    |

---

This completes the full structured reference for the "Transform RSS Feeds into Blog Articles with GPT-4, Human Review & Google Docs" workflow.