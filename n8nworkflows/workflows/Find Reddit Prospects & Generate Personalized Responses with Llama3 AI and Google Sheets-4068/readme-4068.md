Find Reddit Prospects & Generate Personalized Responses with Llama3 AI and Google Sheets

https://n8nworkflows.xyz/workflows/find-reddit-prospects---generate-personalized-responses-with-llama3-ai-and-google-sheets-4068


# Find Reddit Prospects & Generate Personalized Responses with Llama3 AI and Google Sheets

### 1. Workflow Overview

This workflow automates lead generation from Reddit by continuously monitoring target subreddits to discover posts that mention problems your product can solve. It uses AI (Llama3 via Ollama) to analyze the relevance and intent behind each post, filters out irrelevant content, and generates personalized responses to engage potential prospects. All interactions are logged in Google Sheets for tracking and analysis.

The workflow is logically divided into these blocks:

- **1.1 Input Reception:** Manual trigger to start the workflow.
- **1.2 Data Retrieval:** Fetches recent Reddit posts and existing logged posts from Google Sheets.
- **1.3 Data Preparation:** Filters new posts and processes post data for analysis.
- **1.4 AI Analysis & Classification:** Uses Llama3 AI to analyze post content for relevance and intent.
- **1.5 Conditional Routing:** Decides if a post is relevant or irrelevant based on AI output.
- **1.6 Logging & Response:** Logs posts accordingly in Google Sheets and posts personalized Reddit comments on relevant posts.
- **1.7 Iterative Processing:** Processes posts one-by-one to handle batches and looping.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception
- **Overview:** The workflow starts on manual trigger for testing or manual activation.
- **Nodes Involved:**  
  - When clicking ‘Test workflow’
- **Node Details:**
  - Type: Manual Trigger  
  - Configuration: No parameters; user manually triggers the workflow  
  - Inputs: None  
  - Outputs: Initiates the chain to fetch Reddit posts  
  - Potential Failures: None; manual start only

#### 1.2 Data Retrieval
- **Overview:** Retrieves current Reddit posts and existing logged posts from Google Sheets to avoid duplicates.
- **Nodes Involved:**  
  - Get Reddit Posts  
  - Get Existing Posts1
- **Node Details:**

  - **Get Reddit Posts**  
    - Type: Reddit Node  
    - Role: Fetches new posts from specified subreddits using Reddit API credentials  
    - Config: API credentials setup, subreddit(s) specified, post retrieval parameters (e.g., limit, sorting)  
    - Inputs: Triggered by manual or iterative node  
    - Outputs: Raw Reddit post data for further processing  
    - Edge Cases: API rate limits, authentication errors, empty subreddit results

  - **Get Existing Posts1**  
    - Type: Google Sheets Node  
    - Role: Retrieves existing logged posts to detect duplicates  
    - Config: Google account OAuth2 credential, spreadsheet and sheet name specified, read operation  
    - Inputs: Triggered after Reddit posts fetched  
    - Outputs: List of already processed posts  
    - Edge Cases: Authentication failure, sheet access issues

#### 1.3 Data Preparation
- **Overview:** Filters out posts that have already been processed and prepares post data fields for AI processing.
- **Nodes Involved:**  
  - Process Post Fields  
  - Filter New Posts  
  - Process One Post at a Time
- **Node Details:**

  - **Process Post Fields**  
    - Type: Set Node  
    - Role: Normalizes and extracts relevant fields from Reddit post JSON (e.g., post ID, title, body)  
    - Config: Sets key-value pairs for downstream processing  
    - Inputs: Raw Reddit posts  
    - Outputs: Structured post data  
    - Edge Cases: Missing fields in Reddit post JSON

  - **Filter New Posts**  
    - Type: Code Node  
    - Role: Compares new posts with existing logged posts to filter only truly new posts  
    - Config: JavaScript code to check post uniqueness  
    - Inputs: Processed posts and existing posts from Sheets  
    - Outputs: Filtered posts for further processing  
    - Edge Cases: Code exceptions, empty inputs

  - **Process One Post at a Time**  
    - Type: SplitInBatches Node  
    - Role: Processes posts sequentially to avoid rate limits and handle stepwise AI calls  
    - Config: Batch size set to 1  
    - Inputs: Filtered new posts  
    - Outputs: Single post per iteration  
    - Edge Cases: Large batches causing slow processing if batch size >1

#### 1.4 AI Analysis & Classification
- **Overview:** Uses Llama3 AI via Ollama to analyze the content of each Reddit post for relevance and intent.
- **Nodes Involved:**  
  - Ollama Chat Model  
  - AI Relevance Analysis  
  - Parse AI Response
- **Node Details:**

  - **Ollama Chat Model**  
    - Type: Langchain Ollama Chat Model Node  
    - Role: Sends post content to the Llama3 model for natural language understanding  
    - Config: Ollama API credential, Llama3 model selected, prompt template includes post text  
    - Inputs: Single post data from batch processing  
    - Outputs: Raw AI model response  
    - Edge Cases: API downtime, model errors, rate limits

  - **AI Relevance Analysis**  
    - Type: Langchain Agent Node  
    - Role: Executes an agent to analyze AI response, extract intent, sentiment, and relevance classification  
    - Config: Uses AI response as input, agent configured to parse and classify intent  
    - Inputs: Ollama Chat Model output  
    - Outputs: Structured AI analysis result  
    - Edge Cases: Parsing errors, unexpected AI output formats

  - **Parse AI Response**  
    - Type: Code Node  
    - Role: Parses JSON or structured AI response to extract boolean relevance flag and other data  
    - Config: JavaScript extracting fields like `isRelevant`  
    - Inputs: AI Relevance Analysis output  
    - Outputs: Decision flag for conditional routing  
    - Edge Cases: Malformed AI response, JSON parse errors

#### 1.5 Conditional Routing
- **Overview:** Routes posts based on AI relevance decision to either “relevant” or “irrelevant” logging and next steps.
- **Nodes Involved:**  
  - Is Post Relevant?
- **Node Details:**

  - **Is Post Relevant?**  
    - Type: If Node  
    - Role: Evaluates `isRelevant` flag from AI parsing to determine true/false path  
    - Config: Condition checking boolean or string flag  
    - Inputs: Parsed AI response  
    - Outputs: True branch (relevant) and False branch (irrelevant)  
    - Edge Cases: Missing flags, unexpected data types

#### 1.6 Logging & Response
- **Overview:** Logs posts in Google Sheets based on relevance and posts personalized Reddit comments on relevant posts.
- **Nodes Involved:**  
  - Log Relevant Post  
  - Log Irrelevant Post  
  - Post Reddit Comment
- **Node Details:**

  - **Log Relevant Post**  
    - Type: Google Sheets Node  
    - Role: Appends relevant posts and metadata to a designated sheet for tracking leads  
    - Config: Google Sheets OAuth credential, target spreadsheet and sheet, append operation  
    - Inputs: Relevant posts from If node  
    - Outputs: Confirmation or data for next node  
    - Edge Cases: Sheet access errors

  - **Log Irrelevant Post**  
    - Type: Google Sheets Node  
    - Role: Logs posts deemed irrelevant to a separate sheet for record keeping  
    - Config: Similar to Log Relevant Post but different sheet  
    - Inputs: Irrelevant posts from If node  
    - Outputs: Confirmation for continuation  
    - Edge Cases: Same as above

  - **Post Reddit Comment**  
    - Type: Reddit Node  
    - Role: Posts personalized comments generated by AI onto the relevant Reddit posts  
    - Config: Reddit API credential, target post ID, comment body from AI-generated text  
    - Inputs: Triggered after logging relevant post  
    - Outputs: Reddit API response  
    - Edge Cases: Reddit API limits, post locked or deleted, comment rejected

#### 1.7 Iterative Processing
- **Overview:** After handling a post (logging and commenting), the workflow loops back to fetch more posts and repeat processing.
- **Nodes Involved:**  
  - Process One Post at a Time (looping connections)  
- **Node Details:**

  - The SplitInBatches node is connected back to the Get Reddit Posts node and AI Analysis nodes to continue processing the next post in the batch until complete.  
  - Ensures continuous operation over the available new posts.  
  - Edge Cases: Infinite loops if post filtering is incorrect, batch exhaustion

---

### 3. Summary Table

| Node Name               | Node Type                           | Functional Role                         | Input Node(s)                  | Output Node(s)                  | Sticky Note                    |
|-------------------------|-----------------------------------|---------------------------------------|-------------------------------|--------------------------------|-------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger                   | Entry point for manual workflow start | None                          | Get Reddit Posts               |                               |
| Get Reddit Posts        | Reddit Node                       | Fetch new posts from target subreddits| When clicking ‘Test workflow’ | Process Post Fields, Get Existing Posts1 |                               |
| Get Existing Posts1     | Google Sheets Node                | Retrieve logged posts to avoid duplicates | Get Reddit Posts             | Filter New Posts               |                               |
| Process Post Fields     | Set Node                         | Extract relevant post data fields     | Get Reddit Posts              | Filter New Posts               |                               |
| Filter New Posts        | Code Node                        | Filter out already logged posts       | Process Post Fields, Get Existing Posts1 | Process One Post at a Time |                               |
| Process One Post at a Time | SplitInBatches Node              | Process posts sequentially             | Filter New Posts              | Get Reddit Posts, AI Relevance Analysis |                               |
| Ollama Chat Model       | Langchain Ollama Chat Model Node | Send post content to Llama3 AI        | Process One Post at a Time    | AI Relevance Analysis          |                               |
| AI Relevance Analysis   | Langchain Agent Node             | Analyze AI response for relevance and intent | Ollama Chat Model           | Parse AI Response              |                               |
| Parse AI Response       | Code Node                       | Parse AI output to get relevance flag | AI Relevance Analysis         | Is Post Relevant?              |                               |
| Is Post Relevant?       | If Node                        | Route posts based on relevance flag   | Parse AI Response             | Log Relevant Post, Log Irrelevant Post |                               |
| Log Relevant Post       | Google Sheets Node              | Log relevant posts to tracking sheet  | Is Post Relevant? (True)      | Post Reddit Comment            |                               |
| Log Irrelevant Post     | Google Sheets Node              | Log irrelevant posts                   | Is Post Relevant? (False)     | Process One Post at a Time     |                               |
| Post Reddit Comment     | Reddit Node                    | Post personalized comment on Reddit   | Log Relevant Post             | Process One Post at a Time     |                               |
| Sticky Note             | Sticky Note                    | (Empty content)                        | None                         | None                         |                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Purpose: Start workflow manually for testing or manual runs.

2. **Create Reddit Node: Get Reddit Posts**  
   - Type: Reddit  
   - Configure with Reddit API credentials.  
   - Set target subreddit(s) and parameters (e.g., limit to 20 posts, sort by new).  
   - Connect output to both Process Post Fields and Get Existing Posts1 nodes.

3. **Create Google Sheets Node: Get Existing Posts1**  
   - Type: Google Sheets (Read)  
   - Configure OAuth2 credentials for Google account.  
   - Specify spreadsheet and sheet containing logged posts.  
   - Connect output to Filter New Posts.

4. **Create Set Node: Process Post Fields**  
   - Type: Set  
   - Configure to extract and set key fields from the Reddit post JSON such as post ID, title, body, author, link, timestamp.  
   - Connect output to Filter New Posts.

5. **Create Code Node: Filter New Posts**  
   - Type: Code  
   - Write JavaScript code to compare Reddit post IDs against existing logged post IDs from Google Sheets, filtering out duplicates.  
   - Output only new posts.  
   - Connect output to Process One Post at a Time.

6. **Create SplitInBatches Node: Process One Post at a Time**  
   - Type: SplitInBatches  
   - Batch size: 1 (to process posts sequentially).  
   - Connect output to Ollama Chat Model and back to Get Reddit Posts for looping.

7. **Create Langchain Ollama Chat Model Node: Ollama Chat Model**  
   - Type: Langchain Ollama  
   - Configure Ollama API credentials.  
   - Select Llama3 model.  
   - Set prompt template embedding the Reddit post content for AI analysis.  
   - Connect output to AI Relevance Analysis.

8. **Create Langchain Agent Node: AI Relevance Analysis**  
   - Type: Langchain Agent  
   - Configure agent to analyze AI model response for sentiment, intent, and relevance classification.  
   - Connect output to Parse AI Response.

9. **Create Code Node: Parse AI Response**  
   - Type: Code  
   - Parse the AI response JSON to extract a boolean flag `isRelevant`.  
   - Connect output to If Node.

10. **Create If Node: Is Post Relevant?**  
    - Type: If  
    - Condition: Check if `isRelevant` is true.  
    - True branch connects to Log Relevant Post.  
    - False branch connects to Log Irrelevant Post.

11. **Create Google Sheets Node: Log Relevant Post**  
    - Type: Google Sheets (Append)  
    - Configure to append relevant post data and AI results to a “Relevant Posts” sheet.  
    - Connect output to Post Reddit Comment.

12. **Create Google Sheets Node: Log Irrelevant Post**  
    - Type: Google Sheets (Append)  
    - Configure to append irrelevant posts to a separate “Irrelevant Posts” sheet.  
    - Connect output back to Process One Post at a Time to continue processing.

13. **Create Reddit Node: Post Reddit Comment**  
    - Type: Reddit  
    - Configure with Reddit API credentials.  
    - Use AI-generated personalized response as comment body.  
    - Post comment on relevant post ID.  
    - Connect output back to Process One Post at a Time to continue processing next post.

14. **Optional: Add Sticky Note**  
    - Type: Sticky Note  
    - Place on canvas to add workflow comments or instructions.

---

### 5. General Notes & Resources

| Note Content                                                                                                     | Context or Link                                                                                                                  |
|-----------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------|
| This workflow requires Reddit API credentials with appropriate scopes for reading posts and posting comments.  | Reddit developer portal documentation                                                                                           |
| Ollama installation with Llama3 model is mandatory for AI relevance and response generation.                    | https://ollama.ai                                                                                                                |
| Google OAuth2 credentials must have access to Google Sheets API and the specific spreadsheet used for logging.  | Google Cloud Console and Sheets API documentation                                                                               |
| The workflow loops continuously on batch processing for 24/7 monitoring; ensure proper API usage limits are respected. | Reddit and Ollama API rate limits                                                                                               |
| For best results, tailor Llama3 prompt templates in Ollama Chat Model node according to your business context. | Customize prompt to improve AI classification and response personalization                                                      |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.