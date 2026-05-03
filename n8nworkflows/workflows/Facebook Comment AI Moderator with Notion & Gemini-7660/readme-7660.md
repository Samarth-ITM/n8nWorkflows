Facebook Comment AI Moderator with Notion & Gemini

https://n8nworkflows.xyz/workflows/facebook-comment-ai-moderator-with-notion---gemini-7660


# Facebook Comment AI Moderator with Notion & Gemini

### 1. Workflow Overview

This workflow is designed as an intelligent Facebook comment moderator that autonomously reads comments on the latest Facebook post of a page, generates empathetic and contextual replies using AI, and logs processed comments to avoid duplication. It integrates Facebook Graph API, Notion as a knowledge base, and Google Gemini AI via LangChain for natural language processing.

**Target Use Cases:**  
- Automated customer support and engagement on Facebook posts  
- Moderation and timely reply to customer comments with personalized, friendly responses  
- Knowledge-driven responses based on a curated Notion database of products and FAQs  
- Prevention of duplicate replies by tracking processed comments  

**Logical Blocks:**

- **1.1 Input Reception and Scheduling:** Triggered every 10 seconds, fetches latest posts and their comments from Facebook.  
- **1.2 Duplicate Comment Check:** Queries Notion database to verify if a comment has been processed before.  
- **1.3 Knowledge Base Retrieval and Formatting:** Retrieves product info and FAQs from Notion, processes it into a structured searchable format.  
- **1.4 Comment Filtering:** Conditional gate that allows only new comments to proceed.  
- **1.5 AI Processing:** Uses LangChain agent with Google Gemini model to generate a warm, humanized reply based on the comment and knowledge base.  
- **1.6 Reply Posting:** Posts the AI-generated reply back to Facebook as a comment reply.  
- **1.7 Logging:** Saves processed comment IDs into Notion to avoid reprocessing.  

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Scheduling

- **Overview:** This block initiates workflow execution every 10 seconds, fetching all posts of the Facebook page, then isolates the most recent post and fetches its comments.  
- **Nodes Involved:** Schedule Trigger, Posts Fetcher, Last Post Fetcher  
- **Node Details:**  

  - **Schedule Trigger**  
    - Type: Trigger node  
    - Configuration: Runs every 10 seconds (seconds interval = 10)  
    - Input/Output: No input; triggers Posts Fetcher  
    - Edge Cases: Excessive API calls if interval too low; Facebook API rate-limits  

  - **Posts Fetcher**  
    - Type: HTTP Request (Facebook Graph API)  
    - Configuration: GET request to Facebook Graph API endpoint for page posts using predefined Facebook Graph API credentials  
    - URL template uses Facebook Page ID and access token  
    - Input/Output: Triggered by Schedule Trigger; outputs list of posts to Last Post Fetcher  
    - Failure Types: Network errors, token expiration, API limits, malformed requests  

  - **Last Post Fetcher**  
    - Type: HTTP Request (Facebook Graph API)  
    - Configuration: Fetches comments for the most recent post (sorted by creation time descending) using Facebook Graph API  
    - Input/Output: Takes Posts Fetcher output, sorts posts by creation time, extracts latest post ID and fetches its comments  
    - Failure Types: No posts available, API errors, token issues  

---

#### 1.2 Duplicate Comment Check

- **Overview:** Prevents duplicate processing by checking if the latest comment's ID already exists in the Notion "Processed Facebook Comments" database.  
- **Nodes Involved:** CommentID Checker, Merge  
- **Node Details:**  

  - **CommentID Checker**  
    - Type: Notion node (databasePage, getAll)  
    - Configuration: Filters Notion database pages where "Comment ID" matches the latest comment's ID extracted from Last Post Fetcher  
    - Input/Output: Input from Last Post Fetcher; output to Merge node  
    - Failure Types: Notion API errors, invalid filter expressions, empty results  

  - **Merge**  
    - Type: Merge node  
    - Configuration: Merges outputs from CommentID Checker and Knowledge Base nodes for downstream processing  
    - Input/Output: Receives from CommentID Checker and Knowledge Base; outputs to KB Arrange  

---

#### 1.3 Knowledge Base Retrieval and Formatting

- **Overview:** Retrieves all product and FAQ data from a Notion database, then processes it into a structured, searchable format for AI reference.  
- **Nodes Involved:** Knowledge Base, KB Arrange  
- **Node Details:**  

  - **Knowledge Base**  
    - Type: Notion node (databasePage, getAll)  
    - Configuration: Fetches all pages from the specified Notion knowledge base database ID  
    - Input/Output: Input from Last Post Fetcher; output to Merge  
    - Failure Types: Notion API failures, permissions issues  

  - **KB Arrange**  
    - Type: Code node (JavaScript)  
    - Configuration: Processes Notion pages into an array of objects containing product name, price, description, and builds searchable text strings for AI context  
    - Key Expressions: Uses JavaScript to extract and transform properties like rich_text, title, select, number fields  
    - Input/Output: Input from Merge; output to New Comment Conditioner  
    - Failure Types: Unexpected data structure, null or undefined properties, code errors  

---

#### 1.4 Comment Filtering

- **Overview:** Checks if the comment ID exists to determine if the comment is new. The workflow proceeds only if the comment is not previously processed.  
- **Nodes Involved:** New Comment Conditioner  
- **Node Details:**  

  - **New Comment Conditioner**  
    - Type: If node  
    - Configuration: Condition checks if the comment ID property exists (non-empty) to allow passing new comments downstream  
    - Input/Output: Input from KB Arrange; outputs "true" branch to Latest Comment node, else ends workflow  
    - Failure Types: Incorrect property references, empty data, logical errors  

---

#### 1.5 AI Processing

- **Overview:** Generates a personalized and empathetic reply in the target language using LangChain’s AI Agent node powered by Google Gemini. It consults the knowledge base for accurate responses before replying.  
- **Nodes Involved:** Latest Comment, AI Chat Model, AI Agent  
- **Node Details:**  

  - **Latest Comment**  
    - Type: Facebook Graph API node  
    - Configuration: Retrieves full content and metadata of the most recent comment by comment ID  
    - Input/Output: Triggered by New Comment Conditioner; outputs to AI Agent  
    - Failure Types: Comment deleted/hidden, API errors  

  - **AI Chat Model**  
    - Type: LangChain LM Chat Google Gemini node  
    - Configuration: Uses Google Gemini AI model with low temperature (0.2) for stable responses  
    - Input/Output: Input from AI Agent; output back to AI Agent for processing  
    - Failure Types: API quota limits, model errors  

  - **AI Agent**  
    - Type: LangChain Agent node  
    - Configuration: Defines prompt template with company context, tone, language, forbidden expressions, and instructs to consult knowledge base before answering. Uses the latest customer comment as input.  
    - Key Expressions: Utilizes dynamic parameters like [COMPANY_NAME], [TARGET_LANGUAGE], customer comment from Latest Comment, and knowledge base context from KB Arrange  
    - Input/Output: Input from Latest Comment and AI Chat Model; outputs AI-generated reply message to Reply Writer  
    - Failure Types: Prompt errors, missing context, expression parsing failures  

---

#### 1.6 Reply Posting

- **Overview:** Posts the AI-generated reply back to Facebook as a reply to the original comment.  
- **Nodes Involved:** Reply Writer  
- **Node Details:**  

  - **Reply Writer**  
    - Type: Facebook Graph API node (POST request)  
    - Configuration: Posts a comment reply to the original comment ID with message parameter set to AI Agent output  
    - Input/Output: Input from AI Agent; outputs to CommentID to DB node  
    - Failure Types: Posting restrictions, API errors, message too long  

---

#### 1.7 Logging

- **Overview:** Records the comment ID of the processed comment in the Notion "Processed Facebook Comments" database with status "Done" to prevent reprocessing.  
- **Nodes Involved:** CommentID to DB  
- **Node Details:**  

  - **CommentID to DB**  
    - Type: Notion node (databasePage, create)  
    - Configuration: Saves the comment ID as a rich text property and marks status as Done in Notion database  
    - Input/Output: Input from Reply Writer; no output downstream (end of flow)  
    - Failure Types: Notion permission errors, quota limits, invalid property mappings  

---

### 3. Summary Table

| Node Name            | Node Type                             | Functional Role                                   | Input Node(s)           | Output Node(s)          | Sticky Note                                                                                              |
|----------------------|-------------------------------------|--------------------------------------------------|-------------------------|-------------------------|--------------------------------------------------------------------------------------------------------|
| Schedule Trigger      | Schedule Trigger                    | Initiates workflow every 10 seconds               |                         | Posts Fetcher           | Initiates the workflow by retrieving all posts from the specified Facebook Page via the Graph API.     |
| Posts Fetcher        | HTTP Request (Facebook Graph API)   | Fetches posts from Facebook page                   | Schedule Trigger         | Last Post Fetcher       | Processes the list of posts to isolate the most recently published one.                                |
| Last Post Fetcher    | HTTP Request (Facebook Graph API)   | Fetches comments of the latest post                 | Posts Fetcher            | CommentID Checker, Knowledge Base |                                                                                                        |
| CommentID Checker    | Notion (databasePage getAll)         | Checks if comment ID already processed              | Last Post Fetcher        | Merge                   | Queries a database to verify if the latest comment's unique ID has already been processed, preventing duplicate replies. |
| Knowledge Base       | Notion (databasePage getAll)         | Retrieves product and FAQ knowledge base            | Last Post Fetcher        | Merge                   | Fetches the complete, up-to-date product information and knowledge base from the connected Notion database. |
| Merge                | Merge                              | Combines comment check and knowledge base outputs  | CommentID Checker, Knowledge Base | KB Arrange             |                                                                                                        |
| KB Arrange           | Code (JavaScript)                   | Formats Notion data into structured searchable text | Merge                    | New Comment Conditioner  | Transforms the raw knowledge base data into a structured and easily searchable format for the AI model. |
| New Comment Conditioner | If                                | Filters to allow only new comments                   | KB Arrange               | Latest Comment (true), None (false) | A conditional gate; it only allows the workflow to proceed if the comment is new                       |
| Latest Comment       | Facebook Graph API                  | Retrieves full comment details                       | New Comment Conditioner  | AI Agent                | Extracts the full content and metadata of the most recent comment from the latest post.               |
| AI Chat Model        | LangChain LM Chat (Google Gemini)  | Language model for generating replies               | AI Agent (ai_languageModel) | AI Agent                |                                                                                                        |
| AI Agent             | LangChain Agent                   | Generates empathetic, knowledge-based reply         | Latest Comment, AI Chat Model | Reply Writer            | The core processing unit. It analyzes the customer's comment, consults the knowledge base for factual responses, and generates a personalized, empathetic reply in the target language. |
| Reply Writer         | Facebook Graph API (POST)           | Posts AI-generated reply to Facebook comment        | AI Agent                 | CommentID to DB          | Takes the AI-generated response and posts it as a reply to the customer's comment on Facebook.        |
| CommentID to DB      | Notion (databasePage create)         | Logs processed comment ID to Notion database        | Reply Writer             |                         | Logs the ID of the newly processed comment into the database to mark it as complete and prevent reprocessing in the future. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node:**  
   - Type: Schedule Trigger  
   - Set to run every 10 seconds using "secondsInterval" = 10.  

2. **Create Posts Fetcher Node:**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://graph.facebook.com/v21.0/[FACEBOOK_PAGE_ID]/posts?access_token=[FACEBOOK_ACCESS_TOKEN]`  
   - Authentication: Use Facebook Graph API OAuth2 credentials configured with Facebook access token.  
   - Connect Schedule Trigger → Posts Fetcher.  

3. **Create Last Post Fetcher Node:**  
   - Type: HTTP Request  
   - Method: GET  
   - URL expression: `https://graph.facebook.com/v21.0/{{$json["data"][0].id}}/comments?access_token=[FACEBOOK_ACCESS_TOKEN]`  
   - Extract the latest post ID by sorting posts by `created_time` descending in an expression.  
   - Connect Posts Fetcher → Last Post Fetcher.  

4. **Create CommentID Checker Node:**  
   - Type: Notion (databasePage, getAll)  
   - Database ID: `[PROCESSED_COMMENTS_DATABASE_ID]` (Notion database that stores processed comment IDs)  
   - Filter: Where "Comment ID" equals the ID of the latest comment from Last Post Fetcher (use expression).  
   - Connect Last Post Fetcher → CommentID Checker.  
   - Configure Notion credentials with appropriate access rights.  

5. **Create Knowledge Base Node:**  
   - Type: Notion (databasePage, getAll)  
   - Database ID: `[KNOWLEDGE_BASE_DATABASE_ID]` (Notion knowledge base)  
   - Connect Last Post Fetcher → Knowledge Base.  

6. **Create Merge Node:**  
   - Type: Merge (keep all inputs)  
   - Inputs: Connect CommentID Checker and Knowledge Base nodes to Merge.  
   - Connect CommentID Checker → Merge input 0, Knowledge Base → Merge input 1.  

7. **Create KB Arrange Node:**  
   - Type: Code (JavaScript)  
   - Paste the provided JavaScript code to transform raw Notion data into structured knowledge base entries and searchable strings.  
   - Connect Merge → KB Arrange.  

8. **Create New Comment Conditioner Node:**  
   - Type: If node  
   - Condition: Check if comment ID exists (non-empty) in the JSON property from KB Arrange output.  
   - Connect KB Arrange → New Comment Conditioner.  

9. **Create Latest Comment Node:**  
   - Type: Facebook Graph API  
   - Operation: Get comment details by comment ID (from latest comment)  
   - Connect New Comment Conditioner (true branch) → Latest Comment.  

10. **Create AI Chat Model Node:**  
    - Type: LangChain LM Chat Google Gemini  
    - Model Name: `[AI_MODEL_NAME]` (e.g., `gemini-1.5-chat`)  
    - Temperature: 0.2 for stable replies.  
    - Connect AI Agent node’s AI language model input → AI Chat Model → AI Agent.  

11. **Create AI Agent Node:**  
    - Type: LangChain Agent  
    - Configure prompt template with placeholders for company name, type, country, target language, forbidden expressions, preferred expressions, and knowledge base context.  
    - Use expressions to dynamically insert the latest comment message and knowledge base content.  
    - Connect Latest Comment → AI Agent, AI Chat Model → AI Agent (language model input).  

12. **Create Reply Writer Node:**  
    - Type: Facebook Graph API (POST request)  
    - Operation: Post comment reply to the original comment ID with message set to AI Agent’s output.  
    - Connect AI Agent → Reply Writer.  

13. **Create CommentID to DB Node:**  
    - Type: Notion (databasePage, create)  
    - Database ID: `[PROCESSED_COMMENTS_DATABASE_ID]`  
    - Properties:  
      - "Comment ID" (rich_text): Set to current comment ID  
      - "Response Status" (status): Set to "Done"  
    - Connect Reply Writer → CommentID to DB.  

14. **Credential Setup:**  
    - Facebook Graph API OAuth2 credentials with necessary permissions for page posts and comments.  
    - Notion integration token with access to knowledge base and processed comments databases.  
    - LangChain credentials configured for Google Gemini model access.  

15. **Workflow Finalization:**  
    - Ensure proper node connection according to described flow.  
    - Add error handling or retries as needed for API calls.  
    - Test workflow with a test Facebook page and Notion databases.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                          | Context or Link                                                                                      |
|-------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| The workflow uses a LangChain Agent setup with a prompt designed to maintain a warm, unisex, natural tone, avoiding forbidden words. | AI Agent prompt node configuration.                                                                |
| The Facebook Graph API version used is v21.0 for fetching posts and comments, and v23.0 for posting replies and fetching latest comment details. | Facebook Graph API node configurations.                                                             |
| Notion databases must have properties matching the expected schema: "Comment ID" (rich text), "Response Status" (status), and knowledge base properties like product name, price, and description. | Notion database setup instructions.                                                                |
| Facebook access tokens and Notion integration tokens must be kept secure and updated to prevent workflow failures.                  | Security best practices.                                                                             |
| The knowledge base processing script is designed to handle rich_text, title, select, and number property types from Notion.          | Custom code node in KB Arrange.                                                                     |
| Video and community resources for n8n Facebook and Notion integrations can be found on official n8n documentation and forums.       | https://docs.n8n.io/integrations/builtin/nodes/n8n-nodes-base.facebookGraphApi/ and https://docs.n8n.io/integrations/builtin/nodes/n8n-nodes-base.notion/ |

---

**Disclaimer:** The provided content is exclusively derived from an n8n automated workflow. It complies strictly with applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.