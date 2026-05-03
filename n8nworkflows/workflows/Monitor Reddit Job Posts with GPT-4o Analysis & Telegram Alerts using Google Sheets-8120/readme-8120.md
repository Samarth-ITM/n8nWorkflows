Monitor Reddit Job Posts with GPT-4o Analysis & Telegram Alerts using Google Sheets

https://n8nworkflows.xyz/workflows/monitor-reddit-job-posts-with-gpt-4o-analysis---telegram-alerts-using-google-sheets-8120


# Monitor Reddit Job Posts with GPT-4o Analysis & Telegram Alerts using Google Sheets

### 1. Workflow Overview

This workflow automates the monitoring of specific job-related posts on the Reddit subreddit **r/n8n** that are tagged with the flair **"Now Hiring Or Looking For Cofounder"**. It periodically searches Reddit for the newest posts matching this criterion, summarizes each post’s content into a concise sentence using the GPT-4o AI model, stores post metadata along with the summary in a Google Sheets document to avoid duplicates, and finally sends alerts via Telegram with the summarized information.

The workflow is logically divided into the following blocks:

- **1.1 Scheduled Trigger & Reddit Authentication**: Periodically triggers the workflow and authenticates with Reddit to obtain an OAuth2 token.
- **1.2 Reddit Posts Retrieval & Filtering**: Queries Reddit’s API for recent targeted posts and extracts the relevant post data.
- **1.3 Post Processing & Deduplication**: Splits posts into individual items, checks against Google Sheets for duplicates, and filters out already processed posts.
- **1.4 AI Summarization**: Uses an Azure-hosted GPT-4o mini model to generate a short summary sentence for each new post.
- **1.5 Data Persistence & Notification**: Appends new post summaries and metadata to Google Sheets and sends Telegram messages with the summarized content.
- **1.6 Workflow Control & Finalization**: Manages the iterative processing flow and concludes workflow execution.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger & Reddit Authentication

**Overview**:  
This block initiates the workflow every 2 minutes and obtains an OAuth2 access token from Reddit using user credentials and application secrets.

**Nodes Involved**:  
- Schedule Trigger  
- Generate Token - Reddit

**Node Details**:

- **Schedule Trigger**  
  - Type: Schedule Trigger  
  - Role: Periodic workflow initiation every 2 minutes  
  - Config: Fixed interval of 2 minutes, configurable to any desired time  
  - Inputs: None (trigger node)  
  - Outputs: Triggers the next node  
  - Edge Cases: Trigger misconfiguration can cause delayed or missed executions  

- **Generate Token - Reddit**  
  - Type: HTTP Request  
  - Role: Authenticates with Reddit API to obtain OAuth2 bearer token  
  - Config:  
    - POST request to `https://www.reddit.com/api/v1/access_token`  
    - Body: `grant_type=password`, Reddit username, password  
    - Headers: User-Agent set to app name/version; Authorization with base64 encoded client_id:client_secret  
  - Expressions/Variables: Sensitive credentials must be replaced with actual values  
  - Inputs: Trigger from Schedule Trigger  
  - Outputs: JSON response containing `access_token`  
  - Edge Cases: Invalid credentials, rate limiting, network failures, token expiration  
  - Sticky Notes: Advice on generating token and importing cURL command  

---

#### 1.2 Reddit Posts Retrieval & Filtering

**Overview**:  
Uses the valid OAuth2 token to query Reddit’s search API for the newest posts in r/n8n matching the specified flair and extracts the posts array.

**Nodes Involved**:  
- Search N8N  
- Point Out Posts  
- Split Out Code

**Node Details**:

- **Search N8N**  
  - Type: HTTP Request  
  - Role: Queries Reddit API’s search endpoint for posts with flair "Now Hiring Or Looking For Cofounder", sorted by newest, limited to 2 posts  
  - Config:  
    - URL: `https://oauth.reddit.com/r/n8n/search?q=flair:"Now Hiring Or Looking For Cofounder"&restrict_sr=1&sort=new&limit=2`  
    - Headers: Authorization Bearer token from previous node  
  - Inputs: OAuth2 token JSON from "Generate Token - Reddit"  
  - Outputs: JSON response with nested post data in `data.children`  
  - Edge Cases: Token invalidation, Reddit API rate limits, empty search results  
  - Sticky Notes: Option to customize URL, flair, or limit  

- **Point Out Posts**  
  - Type: Set  
  - Role: Extracts the `data.children` array from the Reddit API response and assigns it to a new variable `posts`  
  - Config: Sets `posts` as an array equal to `$json.data.children`  
  - Inputs: Output of "Search N8N"  
  - Outputs: Single JSON item with `posts` array  
  - Edge Cases: Empty or malformed response data  
  - Sticky Notes: Instructs to output only posts and ignore other data  

- **Split Out Code**  
  - Type: Code  
  - Role: Converts the array of posts into separate workflow items, enabling individual processing  
  - Config: JavaScript code loops over `posts` array and returns each post as a separate item  
  - Inputs: Single JSON item with `posts` array  
  - Outputs: Multiple individual items, each representing one Reddit post  
  - Edge Cases: Empty array, malformed post objects  
  - Sticky Notes: Suggests alternative using a 'Split Out' node  

---

#### 1.3 Post Processing & Deduplication

**Overview**:  
Processes each individual Reddit post, checks if it exists already in the Google Sheets database by matching titles, and filters out duplicates.

**Nodes Involved**:  
- Loop Over Items  
- Get Rows That Match  
- If It Doesn't Exist

**Node Details**:

- **Loop Over Items**  
  - Type: SplitInBatches  
  - Role: Processes posts one by one (batch size = 1) to respect rate limits and sequential logic  
  - Config: Default batch size, no special options  
  - Inputs: Multiple post items from "Split Out Code"  
  - Outputs: Single post item per iteration  
  - Edge Cases: Large number of posts may increase processing time  

- **Get Rows That Match**  
  - Type: Google Sheets  
  - Role: Queries Google Sheets to find rows where the column `title` matches the current post’s title  
  - Config:  
    - Document ID and Sheet GID set to a specific Google Sheets file tracking Reddit posts  
    - Filter: `title` column equals current post’s title  
    - OAuth2 credentials configured for Google Sheets access  
  - Inputs: Single post item (title)  
  - Outputs: Zero or more rows matching the title  
  - Edge Cases: Network issues, incorrect document ID, missing sheet permissions  
  - Sticky Notes: Explains the role of sheets as a database for deduplication  

- **If It Doesn't Exist**  
  - Type: If  
  - Role: Conditional node checking if Google Sheets returned no rows (i.e., post is new)  
  - Config: Condition checks if the output from "Get Rows That Match" on `title` is empty  
  - Inputs: Output from "Get Rows That Match"  
  - Outputs:  
    - True branch: Post is new, proceed to AI summarization  
    - False branch: Post already processed, skip and continue loop  
  - Edge Cases: Empty or malformed data, expression errors  

---

#### 1.4 AI Summarization

**Overview**:  
Uses an Azure-hosted GPT-4o mini model to generate a concise summary sentence for each new Reddit post.

**Nodes Involved**:  
- Basic LLM Chain  
- Azure OpenAI Chat Model (chained internally)

**Node Details**:

- **Azure OpenAI Chat Model**  
  - Type: LangChain Azure OpenAI Chat Model  
  - Role: Provides GPT-4o mini language model capabilities for summarization  
  - Config: Uses GPT-4o-mini model hosted on Azure OpenAI service  
  - Credentials: Azure OpenAI API credentials attached  
  - Inputs: Prompt text processed by "Basic LLM Chain"  
  - Outputs: AI-generated completion text  
  - Edge Cases: API quota exceeded, invalid credentials, network errors  
  - Sticky Notes: Suggests customization or use of more complex AI agents  

- **Basic LLM Chain**  
  - Type: LangChain LLM Chain  
  - Role: Defines the prompt and sends the Reddit post content to the AI model for summarization  
  - Config:  
    - Text input: The Reddit post’s selftext field  
    - Prompt: System prompt instructs AI to summarize the post into a short sentence, including the author’s username  
  - Inputs: The current Reddit post JSON item  
  - Outputs: JSON with summarized text in `text` field  
  - Edge Cases: Empty or very short posts, AI response failure, expression errors  

---

#### 1.5 Data Persistence & Notification

**Overview**:  
Appends the new post’s title, summary, and username to Google Sheets and sends a Telegram message to alert about the new post.

**Nodes Involved**:  
- Append Data In Sheet  
- Send a Text Message

**Node Details**:

- **Append Data In Sheet**  
  - Type: Google Sheets  
  - Role: Adds a new row to the Google Sheets tracking document with the post’s title, AI summary, and author username  
  - Config:  
    - Document and Sheet set to same as "Get Rows That Match"  
    - Columns defined explicitly for `title`, `summary`, and `username`  
    - Append mode enabled to add new rows without overwriting  
  - Credentials: Google Sheets OAuth2 credentials (may differ from read credentials)  
  - Inputs: AI summary and Reddit post data  
  - Outputs: Confirmation of append operation  
  - Edge Cases: Permission issues, data type mismatches, API limits  

- **Send a Text Message**  
  - Type: Telegram  
  - Role: Sends a Telegram message with the summary text to a predefined chat ID  
  - Config:  
    - `chatId` set to user’s Telegram chat ID (must be replaced with actual ID)  
    - Message text set to the AI-generated summary  
    - Attribution disabled  
  - Credentials: Telegram Bot API credentials  
  - Inputs: Summary text from Google Sheets append output  
  - Outputs: Telegram API response  
  - Edge Cases: Invalid chat ID, bot not added to chat, Telegram API rate limits  

---

#### 1.6 Workflow Control & Finalization

**Overview**:  
Manages the iterative flow of processing multiple posts and defines workflow endpoint.

**Nodes Involved**:  
- Loop Over Items (second output branch)  
- Finished

**Node Details**:

- **Loop Over Items** (second output)  
  - Role: Loops back to process next post if condition fails (post already exists) or after sending message  
  - Inputs/Outputs: Controls batch iteration of posts  
  - Edge Cases: Infinite loops if misconfigured  

- **Finished**  
  - Type: NoOp  
  - Role: Marks the end of workflow processing for current batch  
  - Inputs: From Loop Over Items when no more posts to process  
  - Outputs: None  
  - Edge Cases: N/A  

---

### 3. Summary Table

| Node Name            | Node Type                     | Functional Role                                | Input Node(s)              | Output Node(s)                 | Sticky Note                                                                                                                        |
|----------------------|-------------------------------|-----------------------------------------------|----------------------------|-------------------------------|-----------------------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger      | Schedule Trigger              | Periodic trigger every 2 minutes               | None                       | Generate Token - Reddit         | Change The Trigger Interval As You Wish                                                                                            |
| Generate Token - Reddit | HTTP Request                | Reddit OAuth2 token generation                  | Schedule Trigger            | Search N8N                    | Generate an Access Token And Use It; Import The cURL Command I Provided Beneath                                                     |
| Search N8N            | HTTP Request                 | Search Reddit for targeted posts                | Generate Token - Reddit     | Point Out Posts               | Customize The URL If You Wish To Change The Limit Or Sub-reddit                                                                    |
| Point Out Posts       | Set                         | Extracts posts array from Reddit response       | Search N8N                 | Split Out Code                | Output Posts Only, Ignore Other Items                                                                                              |
| Split Out Code        | Code                        | Splits posts array into individual items        | Point Out Posts            | Loop Over Items               | Split Objects Into Separate Items, You Can Use The 'Split Out' Node Too                                                            |
| Loop Over Items       | SplitInBatches              | Processes posts one by one                       | Split Out Code             | Finished, Get Rows That Match  |                                                                                                                                   |
| Get Rows That Match   | Google Sheets               | Checks if post title exists in Google Sheets    | Loop Over Items            | If It Doesn't Exist           | About The Google Sheets Nodes: Tracking posts to prevent duplicates                                                                |
| If It Doesn't Exist   | If                          | Filters out posts already processed              | Get Rows That Match        | Basic LLM Chain, Loop Over Items |                                                                                                                                   |
| Basic LLM Chain       | LangChain LLM Chain         | Summarizes post content using AI                 | If It Doesn't Exist        | Append Data In Sheet          | Customize This LLM As You Wish Or Use An Agent If You Want To Add More Complexity                                                    |
| Azure OpenAI Chat Model | LangChain Azure OpenAI Model | Provides GPT-4o mini AI completions              | Basic LLM Chain (internal) | Basic LLM Chain              |                                                                                                                                   |
| Append Data In Sheet  | Google Sheets               | Adds new post summary and metadata to sheet      | Basic LLM Chain            | Send a Text Message           |                                                                                                                                   |
| Send a Text Message   | Telegram                    | Sends Telegram alert with post summary           | Append Data In Sheet       | Loop Over Items               |                                                                                                                                   |
| Finished              | NoOp                        | Marks end of processing batch                     | Loop Over Items            | None                         |                                                                                                                                   |
| Telegram Trigger      | Telegram Trigger (disabled) | Disabled; can be activated to capture chat IDs   | None                      | None                         | Activate This Node And Send Your Bot A Message (Outputs Your Chat ID)                                                              |
| Sticky Notes (various)| Sticky Note                 | Provide instructions, reminders, and references  | N/A                       | N/A                          | See sections above for detailed contents                                                                                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger Node**  
   - Set it to trigger every 2 minutes (or your preferred interval).

2. **Create an HTTP Request Node for Reddit Token Generation**  
   - Name: `Generate Token - Reddit`  
   - Method: POST  
   - URL: `https://www.reddit.com/api/v1/access_token`  
   - Credentials: Use your Reddit app's client ID and secret encoded in the Authorization header as Basic Auth.  
   - Body: Form URL encoded with parameters:  
     - `grant_type`: `password`  
     - `username`: Your Reddit username  
     - `password`: Your Reddit password  
   - Headers:  
     - `User-Agent`: your-app-name/0.1 by YOUR_USERNAME  
   - Connect Schedule Trigger to this node.

3. **Create an HTTP Request Node to Search Reddit**  
   - Name: `Search N8N`  
   - Method: GET  
   - URL: `https://oauth.reddit.com/r/n8n/search?q=flair:"Now Hiring Or Looking For Cofounder"&restrict_sr=1&sort=new&limit=2`  
   - Headers:  
     - `Authorization`: `=bearer {{$json.access_token}}` (expression referencing token from previous node)  
   - Connect `Generate Token - Reddit` to this node.

4. **Create a Set Node to Extract Posts Array**  
   - Name: `Point Out Posts`  
   - Set a new field `posts` with expression: `={{ $json.data.children }}`  
   - Connect `Search N8N` to this node.

5. **Create a Code Node to Split Posts into Individual Items**  
   - Name: `Split Out Code`  
   - JavaScript code:  
     ```javascript
     const postsArray = $input.all()[0].json.posts;
     return postsArray.map(post => ({ json: post }));
     ```  
   - Connect `Point Out Posts` to this node.

6. **Create a SplitInBatches Node to Process Posts Sequentially**  
   - Name: `Loop Over Items`  
   - Default batch size (1)  
   - Connect `Split Out Code` to this node.

7. **Create a Google Sheets Node to Search for Existing Posts**  
   - Name: `Get Rows That Match`  
   - Operation: Lookup rows  
   - Document: Your Google Sheets document that stores posts  
   - Sheet: Sheet containing columns (title, summary, username)  
   - Filter: `title` column equal to current post’s title (`={{ $json.data.title }}`)  
   - Credentials: Google Sheets OAuth2  
   - Connect `Loop Over Items` main output to this node.

8. **Create an If Node to Check Existence**  
   - Name: `If It Doesn't Exist`  
   - Condition: Check if `title` from `Get Rows That Match` is empty (no match found)  
   - Connect `Get Rows That Match` to this node.

9. **Create a LangChain Azure OpenAI Chat Model Node**  
   - Name: `Azure OpenAI Chat Model`  
   - Model: `gpt-4o-mini`  
   - Credentials: Azure OpenAI API  
   - This node is connected internally by the Basic LLM Chain node.

10. **Create a LangChain LLM Chain Node**  
    - Name: `Basic LLM Chain`  
    - Input text: `={{ $('Split Out Code').item.json.data.selftext }}`  
    - Prompt message includes:  
      ```
      You will receive a Reddit post in the category 'Now Hiring' from the user called '{{ $('Split Out Code').item.json.data.author }}'. Your main goal is to turn this Reddit post into a single short sentence.
      ```
    - Connect `If It Doesn't Exist` true output to this node.  
    - Configure the node to use the Azure OpenAI Chat Model as its language model.

11. **Create a Google Sheets Append Node**  
    - Name: `Append Data In Sheet`  
    - Operation: Append row  
    - Document and Sheet: Same as `Get Rows That Match`  
    - Columns to append:  
      - title: `={{ $('Split Out Code').item.json.data.title }}`  
      - summary: `={{ $json.text }}` (output from AI summary)  
      - username: `={{ $('Split Out Code').item.json.data.author }}`  
    - Credentials: Google Sheets OAuth2  
    - Connect `Basic LLM Chain` to this node.

12. **Create a Telegram Node to Send Message**  
    - Name: `Send a Text Message`  
    - Chat ID: Your Telegram chat ID (replace "YOUR_CHAT_ID")  
    - Text: `={{$json.summary}}` (AI summary)  
    - Credentials: Telegram Bot API credentials  
    - Connect `Append Data In Sheet` to this node.

13. **Connect `Send a Text Message` and `If It Doesn't Exist` false output back to `Loop Over Items`**  
    - This completes the loop, processing all posts.

14. **Create a NoOp Node**  
    - Name: `Finished`  
    - Connect `Loop Over Items` first output (no more posts) to this node to mark the workflow end.

15. **Optional**: Create and configure a Telegram Trigger node to capture your chat ID by sending a message to your bot (disabled by default).

---

### 5. General Notes & Resources

| Note Content                                                                                                                         | Context or Link                                                                                                                     |
|--------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------|
| Telegram Bot Token setup instructions: [Telegram Integration Docs](https://docs.n8n.io/integrations/builtin/credentials/telegram/)   | Needed for Telegram notification node credentials                                                                                 |
| Reddit App creation guide: [Reddit Integration Docs](https://docs.n8n.io/integrations/builtin/credentials/reddit/)                   | Create a Reddit script app for OAuth2 credentials                                                                                 |
| Google Sheets OAuth2 integration: [Google Cloud Integration](https://docs.n8n.io/integrations/builtin/credentials/google/oauth-single-service/) | Required for reading and appending data to Google Sheets                                                                          |
| Customize schedule trigger interval as needed                                                                                       | Sticky note on adjusting the trigger frequency                                                                                   |
| cURL command example to obtain Reddit token (replace placeholders):  
```curl -X POST \  
  -u "script_app_id:script_app_secret" \  
  -A "YOUR_APP_NAME/0.1 by YOUR_USERNAME" \  
  -d 'grant_type=password&username=YOUR_USERNAME&password=YOUR_PASSWORD' \  
  https://www.reddit.com/api/v1/access_token``` | Found in sticky notes to import as HTTP Request for token generation                                                              |
| Workflow is designed to avoid duplicate notifications by checking Google Sheets before sending alerts                                | Important to maintain data integrity and prevent spam                                                                             |
| AI summarization prompt can be customized for tone, length, or additional context                                                     | Modify Basic LLM Chain prompt as desired                                                                                         |
| Use the Telegram Trigger node to obtain your chat ID by messaging your bot once activated                                            | Helps to configure correct chat ID for notifications                                                                             |

---

**Disclaimer:** The text provided is exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled are legal and publicly available.