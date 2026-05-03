Convert Telegram Messages to Professional LinkedIn Posts with Gemini AI & Approval Workflow

https://n8nworkflows.xyz/workflows/convert-telegram-messages-to-professional-linkedin-posts-with-gemini-ai---approval-workflow-7466


# Convert Telegram Messages to Professional LinkedIn Posts with Gemini AI & Approval Workflow

---

## 1. Workflow Overview

This workflow automates the transformation of Telegram messages into professional LinkedIn posts using Google Gemini AI, with an integrated approval and editing loop managed via Telegram.

### Purpose and Use Cases

- Convert incoming Telegram messages (containing URLs, topics, or content) from authorized users into LinkedIn posts.
- Enrich content by scraping URLs and searching latest news via Brave Search.
- Use AI (Google Gemini) to generate engaging LinkedIn posts meeting professional standards.
- Provide an interactive Telegram-based approval system allowing users to approve, edit, reject, or schedule posts.
- Log and track all posts and their statuses in Google Sheets for audit and follow-up.

### Logical Blocks

- **1.1 Input Reception & Validation**  
  Triggered by Telegram messages or callback queries, filters authorized users, and generates unique post identifiers.

- **1.2 Content Classification & Routing**  
  Uses AI to classify input into types (URL, Topic, Content, or mixed), extracts URLs and topics, and routes accordingly.

- **1.3 Content Gathering**  
  Scrapes URLs with Firecrawl, conducts Brave Search for topics, and merges all source content.

- **1.4 AI Post Generation**  
  Invokes Google Gemini to create a professional LinkedIn post based on merged content.

- **1.5 User Preview & Approval Workflow**  
  Sends post preview to Telegram with interactive buttons; processes user actions (approve, edit, reject, regenerate, schedule).

- **1.6 Post Publishing and Logging**  
  Publishes approved posts to LinkedIn, updates Google Sheets tracking, and confirms publication via Telegram.

---

## 2. Block-by-Block Analysis

### 1.1 Input Reception & Validation

**Overview:**  
Receives Telegram messages or callback queries, validates the user against a whitelist, generates unique post IDs, and logs the initial request.

**Nodes Involved:**  
- Telegram Trigger  
- Switch  
- Authorized Telegram Users (Set node with whitelist)  
- User Validation (Code node)  
- Authorize Check (If node)  
- Generate Post ID (Code node)  
- Log Initial Request (Google Sheets node)  
- Send a text message (Telegram node for unauthorized users)

**Node Details:**

- **Telegram Trigger**  
  - Type: Trigger node for Telegram messages and callback queries.  
  - Config: Listens for "message" and "callback_query" updates, with image download enabled (extraLarge).  
  - Inputs: None (trigger node).  
  - Outputs: Message or callback data JSON.  
  - Possible failures: Telegram API connectivity, webhook misconfiguration.

- **Switch**  
  - Routes data into two paths: one for normal messages, one for callback queries.  
  - Input: Telegram Trigger output.  
  - Outputs: Message processing or callback handling.

- **Authorized Telegram Users (Set node)**  
  - Stores authorized Telegram user IDs and usernames as arrays.  
  - Used for validation.

- **User Validation (Code node)**  
  - Validates incoming user by Telegram ID or username against the authorized lists.  
  - Outputs user info and authorization status.  
  - Code depends on JSON structure from previous 'Switch' node.  
  - Failure: If input JSON structure changes, code may break.

- **Authorize Check (If node)**  
  - Checks if user is authorized.  
  - If yes: proceeds to Generate Post ID.  
  - If no: sends access denied message.

- **Generate Post ID (Code node)**  
  - Generates a unique post ID combining timestamp and random number.  
  - Adds creation timestamp.  
  - Ensures traceability for each post.

- **Log Initial Request (Google Sheets node)**  
  - Appends or updates a row in a Google Sheet with initial request data including post ID, user info, message text, and timestamp.  
  - Sheet configured to track all posts.

- **Send a text message (Telegram node)**  
  - Replies with "Access Denied" message to unauthorized users.  
  - Uses HTML parse mode and replies to the original message ID.

---

### 1.2 Content Classification & Routing

**Overview:**  
Classifies input content type with AI, extracts URLs and topics, then routes the workflow to appropriate content collection pipelines.

**Nodes Involved:**  
- Intent Categorization (Langchain AI node)  
- Extract the output in JSON Format (Langchain output parser)  
- Parse Classification (Code node)  
- Content Router (If node)  
- If (for topic presence)  
- If1 (for URL presence)  
- Merge (merges multiple paths)

**Node Details:**

- **Intent Categorization (Langchain AI)**  
  - Uses Google Gemini AI with a prompt to classify incoming message into categories: URL, Topic, Content, or mixed variants.  
  - Extracts URLs, topics, direct content, and processing instructions.  
  - Output: JSON with classification and extracted data.  
  - Failure: AI latency, malformed input, or API credential issues.

- **Extract the output in JSON Format (Langchain output parser)**  
  - Parses structured JSON output from the AI node reliably, with auto-fix enabled.  
  - Critical for downstream nodes expecting structured data.

- **Parse Classification (Code node)**  
  - Enhances classification results by adding routing hints (flags for scraping, searching, or direct processing).  
  - Computes processing complexity based on presence of URLs, topics, or content.  
  - Also prepares data for logging and routing.

- **Content Router (If node)**  
  - Routes based on input_type string matching: routes to URL path, topic path, or mixed path.  
  - Controls which content gathering nodes are triggered.

- **If and If1 (If nodes)**  
  - Check presence of topics and URLs respectively to trigger corresponding sub-paths.

- **Merge (Merge node)**  
  - Combines outputs from topic-based search, URL extraction, and direct processing paths for unified downstream processing.

---

### 1.3 Content Gathering

**Overview:**  
Gathers content from URLs via scraping, performs web search for topics, and merges all collected data into a comprehensive brief.

**Nodes Involved:**  
- Extract URLs for Processing (Code node)  
- Loop Over URLs (SplitInBatches node)  
- Scrape a url and get its content (Firecrawl node)  
- Process Scraped Content (Code node)  
- Web Search for the related content (Brave Search node)  
- Merge Content Sources (Code node)

**Node Details:**

- **Extract URLs for Processing (Code node)**  
  - Takes array of URLs and creates individual workflow items for each URL for parallel processing.  
  - Handles empty URL arrays safely.

- **Loop Over URLs (SplitInBatches)**  
  - Processes URLs in batches equal to total URLs count (usually 1 batch) to control parallelism.

- **Scrape a url and get its content (Firecrawl node)**  
  - Scrapes webpage content from each URL, returning article markdown and metadata (e.g., ogUrl).  
  - Requires Firecrawl API credentials.  
  - Failure: Scraping errors, timeouts, blocked URLs.

- **Process Scraped Content (Code node)**  
  - Aggregates scraped content from all URLs, merges with source attributions.  
  - Extracts metadata URLs and handles errors gracefully for missing content.  
  - Outputs merged content body and success flags.

- **Web Search for the related content (Brave Search node)**  
  - Searches latest news and trends based on identified topics.  
  - Returns top search results formatted for AI consumption.  
  - Requires Brave Search API credentials.

- **Merge Content Sources (Code node)**  
  - Combines all gathered data: original message, classification, scraped content, search results.  
  - Constructs a rich context summary to feed into AI generation node.  
  - Determines which content sources were used and builds the final content string with sections (instructions, topics, scraped content, search results).  
  - Provides fallback content prompts if no data was found.  
  - Logs detailed processing metadata for diagnostics.

---

### 1.4 AI Post Generation

**Overview:**  
Generates a professional LinkedIn post using Google Gemini AI based on the merged content brief.

**Nodes Involved:**  
- Initial LinkedIn Post (Langchain AI node)  
- Gemini Model to Generate Post (Langchain AI node - invoked internally)  
- Format Preview (Code node)

**Node Details:**

- **Initial LinkedIn Post (Langchain AI)**  
  - Uses a detailed prompt instructing Gemini AI to generate a LinkedIn post with professional tone, hooks, key points, hashtags, and CTAs.  
  - Input includes merged content from multiple sources.  
  - Output is a strict JSON object with post content and metadata.

- **Gemini Model to Generate Post**  
  - AI model node called by "Initial LinkedIn Post" for processing.

- **Format Preview (Code node)**  
  - Parses Gemini AI JSON response.  
  - Formats a rich preview message with post content, character count, hashtags, hook, and key points for Telegram display.  
  - Handles JSON parsing errors by fallback to raw text.  
  - Adds approval pending flag and timestamp.

---

### 1.5 User Preview & Approval Workflow

**Overview:**  
Sends the generated post preview to Telegram for user review with interactive buttons and handles user actions like approve, edit, reject, or regenerate.

**Nodes Involved:**  
- Send Preview to Telegram (Telegram node)  
- Process Callback (Code node)  
- Answer Query a callback (Telegram node)  
- Route Actions (Switch node)  
- Send Status (Approve) (Telegram node)  
- Send message and wait for response (Telegram node)  
- Send Rejection (Telegram node)  
- Send Rewrite for Approval (Telegram node)  
- Process Rewrite (Code node)  
- Re-Editing Chain (Langchain AI node)  
- Get row(s) in sheet1 (Google Sheets node)  
- Update row in sheet (Google Sheets nodes - multiple variants)

**Node Details:**

- **Send Preview to Telegram**  
  - Sends formatted post preview with inline keyboard buttons: Approve, Edit, Reject.  
  - Uses Telegram callback data encoding post ID and action.  
  - Replies to the original Telegram message.

- **Process Callback (Code node)**  
  - Parses callback query data to identify user action and post ID.  
  - Extracts original post content from the message text.  
  - Prepares structured response with action, user info, and next steps.  
  - Handles actions: approve, edit, reject, regenerate, schedule.  
  - Failure: malformed callback data or missing fields.

- **Answer Query a callback (Telegram node)**  
  - Sends acknowledgement alert to Telegram user that action is processing.

- **Route Actions (Switch node)**  
  - Routes workflow based on action: approve, edit, reject, others.  
  - Controls which downstream nodes execute.

- **Send Status (Approve)**  
  - Notifies user their post was approved and is ready for LinkedIn publishing.

- **Send message and wait for response**  
  - Used when user requests edits; prompts for specific edit instructions via Telegram.

- **Send Rejection**  
  - Informs user of post rejection.

- **Send Rewrite for Approval**  
  - Sends edited/revised post back to user for re-approval with interactive buttons.

- **Process Rewrite (Code node)**  
  - Processes AI-generated rewritten content, updates status to "rewritten," and prepares message for Telegram.

- **Re-Editing Chain (Langchain AI)**  
  - Uses Gemini AI to rewrite post per user instructions, focusing on engagement and professionalism.

- **Get row(s) in sheet1 (Google Sheets)**  
  - Fetches post record from tracking sheet by post ID.

- **Update row in sheet / Update row in sheet1 / Update row in sheet2 / Update row in sheet3 (Google Sheets)**  
  - Various update nodes to track post statuses: approved, rejected, rewritten, with metadata like hashtags, character count, timestamps, etc.

---

### 1.6 Post Publishing and Logging

**Overview:**  
Publishes approved posts to LinkedIn via API, updates tracking sheet, and sends confirmation messages to Telegram.

**Nodes Involved:**  
- Create a post (LinkedIn node)  
- Confirm Publish (Telegram node)  
- Update row in sheet3 (Google Sheets node)

**Node Details:**

- **Create a post (LinkedIn node)**  
  - Publishes the final approved post content to LinkedIn.  
  - Uses OAuth2 credentials, posts publicly.  
  - Failure: API errors, invalid token, content length limits.

- **Confirm Publish (Telegram node)**  
  - Sends Telegram confirmation message including LinkedIn post URL and engagement tracking info.

- **Update row in sheet3 (Google Sheets node)**  
  - Updates tracking sheet with approval status, final post content, and timestamp.

---

## 3. Summary Table

| Node Name                      | Node Type                         | Functional Role                                  | Input Node(s)                  | Output Node(s)                   | Sticky Note                                                                                   |
|--------------------------------|----------------------------------|-------------------------------------------------|-------------------------------|---------------------------------|------------------------------------------------------------------------------------------------|
| Telegram Trigger               | Telegram Trigger                 | Receive Telegram messages & callback queries    | None                          | Switch                         | 🎯 TELEGRAM INPUT & VALIDATION: Receives messages from authorized Telegram users; validates users. |
| Switch                        | Switch                          | Route messages vs callback queries               | Telegram Trigger              | Authorized Telegram Users, Process Callback |                                                                                                |
| Authorized Telegram Users      | Set                            | Store authorized user IDs and usernames          | Switch                       | User Validation                |                                                                                                |
| User Validation               | Code                           | Validate user authorization                       | Authorized Telegram Users     | Authorize Check                |                                                                                                |
| Authorize Check              | If                             | Check user authorization                          | User Validation              | Generate Post ID, Send a text message |                                                                                                |
| Send a text message           | Telegram                       | Send access denied message                        | Authorize Check (unauthorized) | None                          |                                                                                                |
| Generate Post ID              | Code                           | Generate unique post ID                           | Authorize Check (authorized)  | Log Initial Request            |                                                                                                |
| Log Initial Request           | Google Sheets                  | Log initial message details                       | Generate Post ID              | Intent Categorization          |                                                                                                |
| Intent Categorization          | Langchain AI                   | Classify content type, extract URLs & topics     | Log Initial Request           | Extract the output in JSON Format | 🔍 SMART CONTENT ANALYSIS: Uses AI to classify input message and extract key data.             |
| Extract the output in JSON Format | Langchain Output Parser         | Parse AI classification output                     | Intent Categorization          | Parse Classification           |                                                                                                |
| Parse Classification          | Code                           | Enhance classification, add routing hints        | Extract the output in JSON Format | Content Router, Merge         |                                                                                                |
| Content Router               | If                             | Route workflow based on content type              | Parse Classification          | If, If1, Merge                 | ⚡ INTELLIGENT CONTENT ROUTING: Routes content based on classification for efficient processing. |
| If                          | If                             | Check presence of topics                           | Content Router               | Web Search for the related content |                                                                                                |
| If1                         | If                             | Check presence of URLs                             | Content Router               | Extract URLs for Processing    |                                                                                                |
| Extract URLs for Processing   | Code                           | Split URLs into individual items                   | If1                          | Loop Over URLs                 |                                                                                                |
| Loop Over URLs               | SplitInBatches                 | Batch process URLs                                 | Extract URLs for Processing   | Scrape a url and get its content, Process Scraped Content |                                                                                                |
| Scrape a url and get its content | Firecrawl                      | Scrape webpage content                             | Loop Over URLs                | Process Scraped Content        | 📄 CONTENT EXTRACTION: Uses Firecrawl to extract article content per URL.                      |
| Process Scraped Content       | Code                           | Aggregate and merge scraped content               | Loop Over URLs, Scrape        | Merge                        |                                                                                                |
| Web Search for the related content | Brave Search                  | Search latest news based on topics                 | If                          | Merge                        | 📰 REAL-TIME INFORMATION GATHERING: Uses Brave Search for latest industry trends.              |
| Merge                        | Merge                          | Combine URL processing, search results, direct paths | Process Scraped Content, Web Search, Content Router (false route) | Merge Content Sources         |                                                                                                |
| Merge Content Sources        | Code                           | Combine all content sources into comprehensive brief | Merge                        | Initial LinkedIn Post          | ⚙️ MULTI-SOURCE DATA MERGER: Combines original message, scraped content, search results for AI. |
| Initial LinkedIn Post         | Langchain AI                   | Generate LinkedIn post with Gemini AI              | Merge Content Sources         | Format Preview                | ✍️ PROFESSIONAL POST CREATION: Generates professional LinkedIn posts with proper structure.    |
| Format Preview               | Code                           | Parse AI response and format preview for Telegram | Initial LinkedIn Post          | Send Preview to Telegram       | 👀 INTERACTIVE PREVIEW & APPROVAL: Sends preview with analytics and interactive buttons.       |
| Send Preview to Telegram      | Telegram                       | Send formatted post preview with approval buttons | Format Preview                | Update row in sheet            |                                                                                                |
| Process Callback             | Code                           | Parse Telegram callback, extract action details   | Switch (callback path)        | Answer Query a callback, Route Actions | ⚡ USER ACTION HANDLER: Processes approve, edit, reject, regenerate, schedule actions.          |
| Answer Query a callback       | Telegram                       | Acknowledge callback query with alert             | Process Callback             | Route Actions                 |                                                                                                |
| Route Actions                | Switch                        | Route workflow based on user action                | Answer Query a callback       | Send Status (Approve), Send message and wait for response, Send Rejection |                                                                                                |
| Send Status (Approve)         | Telegram                       | Inform user post approved                          | Route Actions                | Get row(s) in sheet1          |                                                                                                |
| Get row(s) in sheet1          | Google Sheets                  | Retrieve post details for publishing               | Send Status (Approve)         | Create a post                 |                                                                                                |
| Create a post                | LinkedIn                      | Publish post on LinkedIn                            | Get row(s) in sheet1          | Confirm Publish               | 📤 LINKEDIN POST PUBLICATION: Publishes approved post and confirms success.                    |
| Confirm Publish              | Telegram                       | Notify user of successful LinkedIn post            | Create a post                | Update row in sheet3          |                                                                                                |
| Update row in sheet          | Google Sheets                  | Track post metadata after preview                   | Send Preview to Telegram      | None                         |                                                                                                |
| Update row in sheet1         | Google Sheets                  | Update post with rewritten content                  | Process Rewrite              | Send Rewrite for Approval     |                                                                                                |
| Update row in sheet2         | Google Sheets                  | Mark post as rejected                               | Send Rejection               | None                         |                                                                                                |
| Update row in sheet3         | Google Sheets                  | Mark post as approved                               | Confirm Publish              | None                         |                                                                                                |
| Send Rewrite for Approval     | Telegram                       | Send rewritten post for user re-approval           | Update row in sheet1          | None                         | 🔧 AI-POWERED CONTENT REFINEMENT: Shows improved content for re-approval or further editing.    |
| Send message and wait for response | Telegram                       | Prompt user for edit instructions                   | Route Actions (edit path)     | Re-Editing Chain             |                                                                                                |
| Process Rewrite             | Code                           | Process rewritten AI content and prepare for approval | Re-Editing Chain            | Update row in sheet1          |                                                                                                |
| Re-Editing Chain             | Langchain AI                   | AI rewrite of post per user instructions            | Send message and wait for response | Process Rewrite           |                                                                                                |

---

## 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger node**  
   - Type: Telegram Trigger  
   - Configure to listen for "message" and "callback_query" updates. Enable download of images (extraLarge).  
   - Connect to a Switch node.

2. **Add Switch node**  
   - Configure to check if incoming data contains "message" or "callback_query".  
   - Route "message" to authorized user validation path, "callback_query" to action processing path.

3. **Authorized Telegram Users (Set node)**  
   - Create JSON with arrays for authorized telegram_ids and usernames.  
   - Connect Switch "message" output to this node.

4. **User Validation (Code node)**  
   - Validate user against authorized lists using code that checks Telegram message sender ID or username.  
   - Output includes authorization flag and user info.  
   - Connect from Authorized Telegram Users node.

5. **Authorize Check (If node)**  
   - Check if user is authorized (authorization flag true).  
   - If yes, proceed to generate post ID; if no, send "Access Denied" message.

6. **Send a text message (Telegram node)**  
   - Send "Access Denied" message with HTML formatting to unauthorized users.  
   - Connect from Authorize Check "no" output.

7. **Generate Post ID (Code node)**  
   - Generate unique post ID based on ISO timestamp and random number.  
   - Attach creation timestamp.  
   - Connect from Authorize Check "yes" output.

8. **Log Initial Request (Google Sheets node)**  
   - Append or update row in Google Sheet with post ID, user info, message text, and timestamp.  
   - Use Google Sheets OAuth2 credentials.  
   - Connect from Generate Post ID.

9. **Intent Categorization (Langchain AI node)**  
   - Use Google Gemini AI with prompt to classify message into categories (URL, Topic, Content, mixed).  
   - Extract URLs, topics, direct content, and instructions.  
   - Connect from Log Initial Request.

10. **Extract the output in JSON Format (Langchain Output Parser)**  
    - Parse the AI output into strict JSON.  
    - Connect from Intent Categorization.

11. **Parse Classification (Code node)**  
    - Add flags for routing, count URLs and topics, determine complexity.  
    - Prepare data for routing and logging.  
    - Connect from Extract the output in JSON Format.

12. **Content Router (If node)**  
    - Define conditions to route based on input_type property (URL, Topic, URL+Topic).  
    - Connect from Parse Classification.

13. **If (topics present)**  
    - Check if topics array is not empty.  
    - Connect from Content Router "true" branch.

14. **If1 (URLs present)**  
    - Check if URLs array is not empty.  
    - Connect from Content Router "true" branch.

15. **Web Search for the related content (Brave Search node)**  
    - Configure to search latest news with query based on topics.  
    - Use Brave Search API credentials.  
    - Connect from If (topics present).

16. **Extract URLs for Processing (Code node)**  
    - Split URLs into individual items for scraping.  
    - Connect from If1 (URLs present).

17. **Loop Over URLs (SplitInBatches)**  
    - Batch process URLs (batch size = total URLs).  
    - Connect from Extract URLs for Processing.

18. **Scrape a url and get its content (Firecrawl node)**  
    - Scrape content from each URL using Firecrawl API.  
    - Connect from Loop Over URLs.

19. **Process Scraped Content (Code node)**  
    - Merge all scraped content with source attribution.  
    - Connect from Loop Over URLs and Scrape a url and get its content.

20. **Merge (Merge node)**  
    - Merge outputs from:  
      - Web Search for the related content  
      - Process Scraped Content  
      - Content Router false route (direct content)  
    - Connect all three to merge inputs.

21. **Merge Content Sources (Code node)**  
    - Combine original message, scraped content, search results, metadata into final content brief.  
    - Build context summary for AI generation.  
    - Connect from Merge node.

22. **Initial LinkedIn Post (Langchain AI node)**  
    - Use Google Gemini AI to create a LinkedIn post with professional tone and metadata.  
    - Connect from Merge Content Sources.

23. **Format Preview (Code node)**  
    - Parse AI JSON output, format detailed preview text for Telegram, including character count, hashtags, key points.  
    - Connect from Initial LinkedIn Post.

24. **Send Preview to Telegram (Telegram node)**  
    - Send preview text with inline keyboard buttons: Approve, Edit, Reject.  
    - Configure callback data with action and post ID.  
    - Connect from Format Preview.

25. **Update row in sheet (Google Sheets node)**  
    - Update Google Sheet with initial generated post, hashtags, character count, and content sources.  
    - Connect from Send Preview to Telegram.

26. **Process Callback (Code node)**  
    - Parse Telegram callback query, extract action (approve, edit, reject, regenerate, schedule), user info, and post content.  
    - Prepare structured response for workflow routing.  
    - Connect from Switch "callback_query" output.

27. **Answer Query a callback (Telegram node)**  
    - Send processing alert to user for callback actions.  
    - Connect from Process Callback.

28. **Route Actions (Switch node)**  
    - Route workflow to separate paths based on user action.  
    - Connect from Answer Query a callback.

29. **Send Status (Approve) (Telegram node)**  
    - Notify of approval and readiness for LinkedIn publishing.  
    - Connect from Route Actions (approve).

30. **Get row(s) in sheet1 (Google Sheets node)**  
    - Retrieve post record by post ID.  
    - Connect from Send Status (Approve).

31. **Create a post (LinkedIn node)**  
    - Publish post content to LinkedIn using OAuth2 credentials.  
    - Connect from Get row(s) in sheet1.

32. **Confirm Publish (Telegram node)**  
    - Notify user of successful LinkedIn post publication.  
    - Connect from Create a post.

33. **Update row in sheet3 (Google Sheets node)**  
    - Update sheet with approval status and final post metadata.  
    - Connect from Confirm Publish.

34. **Send message and wait for response (Telegram node)**  
    - Ask user for edit instructions when "edit" action selected.  
    - Connect from Route Actions (edit).

35. **Re-Editing Chain (Langchain AI node)**  
    - Use Gemini AI to rewrite post based on user instructions.  
    - Connect from Send message and wait for response.

36. **Process Rewrite (Code node)**  
    - Process rewritten content and prepare message for Telegram.  
    - Connect from Re-Editing Chain.

37. **Update row in sheet1 (Google Sheets node)**  
    - Update post record with rewritten content.  
    - Connect from Process Rewrite.

38. **Send Rewrite for Approval (Telegram node)**  
    - Send rewritten post back to user for re-approval.  
    - Connect from Update row in sheet1.

39. **Send Rejection (Telegram node)**  
    - Inform user of rejection.  
    - Connect from Route Actions (reject).

40. **Update row in sheet2 (Google Sheets node)**  
    - Mark post as rejected in tracking sheet.  
    - Connect from Send Rejection.

---

## 5. General Notes & Resources

| Note Content                                                                                                                                                            | Context or Link                                                                                      |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| 🎯 TELEGRAM INPUT & VALIDATION: Ensures only authorized Telegram users can submit content, improving security and control.                                           | Sticky Note near Telegram Trigger and User Validation nodes                                        |
| 🔍 SMART CONTENT ANALYSIS: AI-driven classification distinguishes content types to route processing efficiently.                                                     | Sticky Note near Intent Categorization and Parse Classification nodes                              |
| ⚡ INTELLIGENT CONTENT ROUTING: Routes content to scraping, searching, or direct processing pipelines based on classification.                                         | Sticky Note near Content Router and related nodes                                                  |
| 📄 CONTENT EXTRACTION: Uses Firecrawl to extract rich article text and metadata from URLs, supporting multi-URL batch processing.                                   | Sticky Note near Scrape a url and get its content node                                             |
| 📰 REAL-TIME INFORMATION GATHERING: Brave Search fetches latest news and trending articles to enhance post context and relevance.                                    | Sticky Note near Web Search for the related content node                                          |
| ⚙️ MULTI-SOURCE DATA MERGER: Combines original message, scraped content, search results, and metadata into a comprehensive brief for AI post generation.             | Sticky Note near Merge Content Sources node                                                       |
| ✍️ PROFESSIONAL POST CREATION: Google Gemini AI generates LinkedIn posts with professional tone, proper formatting, and strategic hashtags.                        | Sticky Note near Initial LinkedIn Post node                                                       |
| 👀 INTERACTIVE PREVIEW & APPROVAL: Telegram preview with analytics and interactive buttons empowers user control over final post content.                            | Sticky Note near Format Preview and Send Preview to Telegram nodes                                |
| ⚡ USER ACTION HANDLER: Handles user interactions such as approve, edit, reject, regenerate, and schedule seamlessly within Telegram.                                | Sticky Note near Process Callback and Route Actions nodes                                         |
| 🔧 AI-POWERED CONTENT REFINEMENT: Iterative AI rewriting based on user instructions ensures high-quality, engaging LinkedIn posts.                                   | Sticky Note near Re-Editing Chain and Process Rewrite nodes                                       |
| 📤 LINKEDIN POST PUBLICATION: Posts content directly to LinkedIn via API, confirms success, and updates tracking for accountability.                                | Sticky Note near Create a post and Confirm Publish nodes                                          |
| Workflow uses Google Sheets extensively for tracking posts, statuses, and metadata, ensuring auditability and process transparency.                                  | Google Sheets nodes throughout the workflow                                                       |
| Requires multiple API credentials: Telegram Bot API, Google Gemini (PaLM), Firecrawl, Brave Search, LinkedIn OAuth2, and Google Sheets OAuth2.                       | Credential setup required for all external APIs                                                   |

---

(disclaimer: The provided text originates exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.)

---