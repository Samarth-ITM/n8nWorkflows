Translate & Repost Twitter Threads in Multiple Languages with OpenAI

https://n8nworkflows.xyz/workflows/translate---repost-twitter-threads-in-multiple-languages-with-openai-4233


# Translate & Repost Twitter Threads in Multiple Languages with OpenAI

### 1. Workflow Overview

This workflow is designed to translate and repost Twitter threads in multiple languages using OpenAI's language models. It is targeted at users who want to automatically translate entire Twitter threads and repost them on Twitter, potentially in different languages, while managing Twitter authentication and session states.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception & Configuration:** Receives manual or external triggers with configuration parameters such as target language and tweet URL.
- **1.2 Tweet Extraction & Processing:** Extracts the original tweet and its replies to reconstruct the full thread.
- **1.3 Translation & Rewriting:** Uses OpenAI's language models to translate and optionally rewrite tweets in the target language.
- **1.4 Twitter Authentication & Session Management:** Handles login, including two-factor authentication (2FA), session retrieval, and updates.
- **1.5 Tweet Posting Loop:** Posts the translated tweets back to Twitter sequentially, managing first tweet and replies, and handling posting errors.
- **1.6 Error Handling & Workflow Control:** Manages authorization errors, login failures, and iteration control within the loop.

---

### 2. Block-by-Block Analysis

---

#### 1.1 Input Reception & Configuration

**Overview:**  
This block initializes the workflow by receiving manual or external triggers that specify the language and Twitter thread URL. It sets up necessary configuration parameters.

**Nodes Involved:**  
- When clicking ‘Test workflow’ (Manual Trigger)  
- config (Code)  
- Execute Workflow (Execute Workflow)  
- When Executed by Another Workflow (Execute Workflow Trigger)

**Node Details:**  

- **When clicking ‘Test workflow’**  
  - Type: Manual Trigger  
  - Role: Entry point for manual testing with predefined input (language, tweet URL).  
  - Connections: Outputs to `config`.  
  - Edge cases: Missing or malformed input parameters.

- **config**  
  - Type: Code  
  - Role: Prepare configuration variables for downstream nodes.  
  - Connections: Outputs to `Execute Workflow`.  
  - Edge cases: Errors in code logic or undefined variables.

- **Execute Workflow**  
  - Type: Execute Workflow  
  - Role: Calls the main sub-workflow that performs translation and posting logic.  
  - Connections: Inputs from `config`, outputs to `Extract tweets list`.  
  - Edge cases: Sub-workflow failures or misconfiguration.

- **When Executed by Another Workflow**  
  - Type: Execute Workflow Trigger  
  - Role: Alternative entry point when called by another workflow, receives input parameters.  
  - Connections: Outputs to `Extract Tweet ID and Username`.  
  - Edge cases: Input parameter validation.

---

#### 1.2 Tweet Extraction & Processing

**Overview:**  
This block extracts the original tweet and all related replies that form the thread. It processes and merges tweet data to prepare for translation.

**Nodes Involved:**  
- Extract Tweet ID and Username (Function)  
- Get first tweet (HTTP Request)  
- Extract Conversation and Author ID (Function)  
- Merge all tweet infos (Merge)  
- Get Tweet Replies (HTTP Request)  
- Fetch tweets which are connected to first tweet (Code)  
- Merge first tweet and others (Merge)  
- Filter empty ones (Filter)  
- Extract tweets list (Code)  
- MAIN_LOOP (Split In Batches)  
- DONE (No Operation)  

**Node Details:**  

- **Extract Tweet ID and Username**  
  - Type: Function  
  - Role: Parses input URL to extract tweet ID and username for API calls.  
  - Inputs: Trigger input (tweet URL).  
  - Outputs: Tweet ID and username JSON for next node.  
  - Edge cases: Invalid URL format, missing tweet ID.

- **Get first tweet**  
  - Type: HTTP Request  
  - Role: Fetches the original tweet data using Twitter API.  
  - Inputs: Tweet ID from previous node.  
  - Outputs: Raw tweet data.  
  - Edge cases: Twitter API rate limits, network errors, invalid tweet ID, authorization errors.

- **Extract Conversation and Author ID**  
  - Type: Function  
  - Role: Extracts conversation ID and author ID needed for fetching replies.  
  - Inputs: First tweet data.  
  - Outputs: Conversation and author IDs.  
  - Edge cases: Missing fields in tweet data.

- **Merge all tweet infos**  
  - Type: Merge  
  - Role: Combines first tweet and extracted IDs into a single dataset.  
  - Inputs: From `Extract Conversation and Author ID` and `Get first tweet`.  
  - Outputs: Combined tweet info.  

- **Get Tweet Replies**  
  - Type: HTTP Request  
  - Role: Retrieves replies to the original tweet to form the thread.  
  - Inputs: Conversation ID, author ID.  
  - Outputs: Replies data.  
  - Edge cases: API limits, empty replies, authorization errors.

- **Fetch tweets which are connected to first tweet**  
  - Type: Code  
  - Role: Processes replies to filter and format them as part of the thread.  
  - Inputs: Replies data.  
  - Outputs: Formatted list of tweets.  
  - Edge cases: Empty reply lists, code exceptions.

- **Merge first tweet and others**  
  - Type: Merge  
  - Role: Combines original tweet and replies into a full thread dataset.  
  - Inputs: Processed replies and first tweet data.  
  - Outputs: Complete thread.  

- **Filter empty ones**  
  - Type: Filter  
  - Role: Removes any empty or invalid tweets from the list.  
  - Inputs: Thread dataset.  
  - Outputs: Cleaned tweet list for translation.  
  - Edge cases: All tweets filtered out (empty thread).

- **Extract tweets list**  
  - Type: Code  
  - Role: Final preparation of tweet list for batch processing.  
  - Inputs: From `Execute Workflow`.  
  - Outputs: List to be processed in batches.  

- **MAIN_LOOP**  
  - Type: Split In Batches  
  - Role: Iterates over tweets one at a time for translation and posting.  
  - Inputs: Tweet list.  
  - Outputs: Single tweet per execution cycle.  
  - Edge cases: Empty batches, batch size configuration.

- **DONE**  
  - Type: No Operation  
  - Role: Marks end of extraction and processing sequence.

---

#### 1.3 Translation & Rewriting

**Overview:**  
This block uses OpenAI language models to translate tweets into the target language and optionally rewrite them for clarity or style.

**Nodes Involved:**  
- Translator_openai (Langchain Agent)  
- OpenAI Chat Model (Langchain LM Chat OpenAI)  
- Rewriter (Langchain Agent)  

**Node Details:**  

- **OpenAI Chat Model**  
  - Type: Langchain LM Chat OpenAI  
  - Role: Provides OpenAI GPT chat-based language model for translation and rewriting tasks.  
  - Configuration: Uses OpenAI API credentials with chat model parameters.  
  - Inputs: Prompt with tweet text and target language.  
  - Outputs: Translated or rewritten tweet text.  
  - Edge cases: API quota limits, invalid parameters, network issues.

- **Translator_openai**  
  - Type: Langchain Agent  
  - Role: Orchestrates translation process using OpenAI chat model.  
  - Inputs: Single tweet text from batch.  
  - Outputs: Translated tweet text.  

- **Rewriter**  
  - Type: Langchain Agent  
  - Role: Optionally rewrites translated text to improve fluency or style.  
  - Inputs: Output from Translator_openai.  
  - Outputs: Final text for posting.  

---

#### 1.4 Twitter Authentication & Session Management

**Overview:**  
This block manages Twitter user authentication, including handling two-factor authentication (2FA), session retrieval, and updating stored session information for posting tweets.

**Nodes Involved:**  
- Get twitter session (Notion)  
- Check session data (If)  
- Login Twitter (HTTP Request)  
- Check login status (If)  
- login_2fa error (If)  
- Wait for 2fa code (Wait)  
- Login Twitter with 2fa (HTTP Request)  
- Check login (If)  
- Update twitter session (Notion)  
- Authorization Error (If)  

**Node Details:**  

- **Get twitter session**  
  - Type: Notion  
  - Role: Retrieves stored Twitter session data from Notion database.  
  - Inputs: Trigger from login status check.  
  - Outputs: Session info or empty.  
  - Edge cases: Notion API errors, missing session data.

- **Check session data**  
  - Type: If  
  - Role: Verifies whether valid session data exists.  
  - Outputs: Proceeds with posting if valid, else triggers login.  

- **Login Twitter**  
  - Type: HTTP Request  
  - Role: Initiates login request to Twitter API with credentials.  
  - Outputs: Status of login attempt.  
  - Edge cases: Incorrect credentials, network failure.

- **Check login status**  
  - Type: If  
  - Role: Checks if login was successful or requires 2FA.  
  - Outputs: Proceeds to session retrieval or 2FA workflow.

- **login_2fa error**  
  - Type: If  
  - Role: Detects 2FA requirement or login error.  
  - Outputs: Either triggers 2FA wait or retries login.

- **Wait for 2fa code**  
  - Type: Wait  
  - Role: Pauses workflow until 2FA code is entered via webhook.  
  - Edge cases: Timeout or no code entered.

- **Login Twitter with 2fa**  
  - Type: HTTP Request  
  - Role: Submits 2FA code and completes login.  

- **Check login**  
  - Type: If  
  - Role: Validates login result after 2FA.  
  - Outputs: Proceeds or retries login.

- **Update twitter session**  
  - Type: Notion  
  - Role: Stores updated session data back to Notion.  

- **Authorization Error**  
  - Type: If  
  - Role: Detects authorization failure post-login, triggers re-login or error handling.

---

#### 1.5 Tweet Posting Loop

**Overview:**  
This block posts the translated tweets back to Twitter sequentially, handling the first tweet differently from replies, checking posting success, and managing iteration state.

**Nodes Involved:**  
- First tweet or thread (If)  
- Set tweet infos (Set)  
- Post first tweet (HTTP Request)  
- Check posting (If)  
- Post reply tweet (HTTP Request)  
- Check posting1 (If)  
- Authorization Error1 (If)  
- Set first tweet id and session (Set)  
- set_iteration_outputs (Code)  

**Node Details:**  

- **First tweet or thread**  
  - Type: If  
  - Role: Determines if current tweet is the first in thread or a reply.  
  - Outputs: Branches to posting first tweet or reply.

- **Set tweet infos**  
  - Type: Set  
  - Role: Prepares tweet content and metadata for posting.  

- **Post first tweet**  
  - Type: HTTP Request  
  - Role: Posts the initial tweet of the thread to Twitter API.  
  - Edge cases: API rate limits, content validation errors.

- **Check posting**  
  - Type: If  
  - Role: Confirms if the first tweet posted successfully.  
  - Outputs: Continues or triggers authorization error handling.

- **Post reply tweet**  
  - Type: HTTP Request  
  - Role: Posts replies referencing the first tweet ID to maintain thread structure.  
  - Edge cases: Reply limits, invalid references.

- **Check posting1**  
  - Type: If  
  - Role: Confirms if reply tweet posted successfully.

- **Authorization Error1**  
  - Type: If  
  - Role: Handles posting authorization errors, may trigger session refresh.

- **Set first tweet id and session**  
  - Type: Set  
  - Role: Stores the ID of the first posted tweet and session info for replies.  

- **set_iteration_outputs**  
  - Type: Code  
  - Role: Manages batch iteration state for looping through tweets.  

---

#### 1.6 Error Handling & Workflow Control

**Overview:**  
Handles errors such as authorization failures and manages workflow iteration and termination signals.

**Nodes Involved:**  
- Authorization Error  
- Authorization Error1  
- Send error message (No Operation)  
- No Operation, do nothing (No Operation)  
- DONE (No Operation)  

**Node Details:**  

- **Authorization Error / Authorization Error1**  
  - Type: If  
  - Role: Detects authorization failures and triggers login retry or error notification.  

- **Send error message**  
  - Type: No Operation  
  - Role: Placeholder for sending error notification or logging.

- **No Operation, do nothing**  
  - Type: No Operation  
  - Role: Ends branches where no further action is needed.

- **DONE**  
  - Type: No Operation  
  - Role: Marks completion of batch processing.

---

### 3. Summary Table

| Node Name                     | Node Type                      | Functional Role                      | Input Node(s)                 | Output Node(s)                       | Sticky Note                                                                                                         |
|-------------------------------|-------------------------------|------------------------------------|------------------------------|------------------------------------|---------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’  | Manual Trigger                | Entry point for manual test        |                              | config                             |                                                                                                                     |
| config                        | Code                          | Sets up configuration variables    | When clicking ‘Test workflow’ | Execute Workflow                   |                                                                                                                     |
| Execute Workflow              | Execute Workflow              | Calls main processing sub-workflow | config                       | Extract tweets list                |                                                                                                                     |
| Extract tweets list           | Code                          | Prepares tweet list for batches    | Execute Workflow             | MAIN_LOOP                        |                                                                                                                     |
| MAIN_LOOP                    | Split In Batches              | Iterates over tweets for translation/posting | Extract tweets list          | DONE, Translator_openai            |                                                                                                                     |
| DONE                         | No Operation                 | Marks end of processing             | MAIN_LOOP                   | REMOVE THIS NODE                  |                                                                                                                     |
| Translator_openai             | Langchain Agent              | Translates tweets                  | MAIN_LOOP                   | Rewriter                         |                                                                                                                     |
| OpenAI Chat Model            | Langchain LM Chat OpenAI     | Underlying OpenAI model            | Translator_openai, Rewriter  | Translator_openai, Rewriter       |                                                                                                                     |
| Rewriter                     | Langchain Agent              | Rewrites translated tweets         | Translator_openai            | Get twitter session               |                                                                                                                     |
| Get twitter session          | Notion                       | Retrieves stored Twitter session   | Check login status           | Check session data                |                                                                                                                     |
| Check session data           | If                           | Checks if session exists            | Get twitter session          | First tweet or thread, Login Twitter |                                                                                                                     |
| Login Twitter                | HTTP Request                 | Performs Twitter login              | Check session data, Authorization Error | Check login status, Authorization Error |                                                                                                                     |
| Check login status           | If                           | Checks login success or 2FA needed | Login Twitter                | Get twitter session, login_2fa error |                                                                                                                     |
| login_2fa error             | If                           | Detects 2FA requirement             | Check login status           | Login Twitter with 2fa, Wait for 2fa code |                                                                                                                     |
| Wait for 2fa code            | Wait                         | Waits for user input of 2FA code   | login_2fa error              | Login Twitter with 2fa            |                                                                                                                     |
| Login Twitter with 2fa        | HTTP Request                 | Submits 2FA code and logs in       | Wait for 2fa code, login_2fa error | Check login                      |                                                                                                                     |
| Check login                  | If                           | Validates login post-2FA            | Login Twitter with 2fa       | Update twitter session, Send error message |                                                                                                                     |
| Update twitter session       | Notion                       | Saves updated Twitter session      | Check login                 | Get twitter session               |                                                                                                                     |
| Authorization Error          | If                           | Detects authorization failure      | Check posting                | Login Twitter                    |                                                                                                                     |
| Authorization Error1         | If                           | Detects authorization failure      | Check posting1               | Get twitter session              |                                                                                                                     |
| First tweet or thread        | If                           | Distinguishes first tweet from replies | Check session data          | Set tweet infos, Set first tweet id and session |                                                                                                                     |
| Set tweet infos              | Set                          | Prepares tweet data for posting    | First tweet or thread        | Post reply tweet                 |                                                                                                                     |
| Set first tweet id and session | Set                         | Stores first tweet ID and session  | First tweet or thread        | Post first tweet                 |                                                                                                                     |
| Post first tweet             | HTTP Request                 | Posts initial tweet                 | Set first tweet id and session | Check posting                   |                                                                                                                     |
| Check posting                | If                           | Checks if first tweet posted       | Post first tweet             | set_iteration_outputs, Authorization Error |                                                                                                                     |
| Post reply tweet            | HTTP Request                 | Posts reply tweet in thread        | Set tweet infos              | Check posting1                  |                                                                                                                     |
| Check posting1               | If                           | Checks if reply tweet posted       | Post reply tweet             | set_iteration_outputs, Authorization Error1 |                                                                                                                     |
| set_iteration_outputs        | Code                         | Manages batch iteration state      | Check posting, Check posting1 | MAIN_LOOP                      |                                                                                                                     |
| Extract Tweet ID and Username | Function                    | Extracts tweet ID and username     | When Executed by Another Workflow | Get first tweet, Merge all tweet infos |                                                                                                                     |
| Get first tweet              | HTTP Request                 | Retrieves original tweet data      | Extract Tweet ID and Username | Extract Conversation and Author ID, Merge first tweet and others |                                                                                                                     |
| Extract Conversation and Author ID | Function                | Extracts conversation and author IDs | Get first tweet              | Merge all tweet infos             |                                                                                                                     |
| Merge all tweet infos        | Merge                        | Combines tweet info data           | Extract Conversation and Author ID, Extract Tweet ID and Username | Get Tweet Replies               |                                                                                                                     |
| Get Tweet Replies            | HTTP Request                 | Fetches replies to first tweet     | Merge all tweet infos        | Fetch tweets which are connected to first tweet |                                                                                                                     |
| Fetch tweets which are connected to first tweet | Code               | Processes and filters replies      | Get Tweet Replies            | Merge first tweet and others      |                                                                                                                     |
| Merge first tweet and others | Merge                        | Combines original tweet and replies | Fetch tweets which are connected to first tweet, Get first tweet | Filter empty ones               |                                                                                                                     |
| Filter empty ones            | Filter                       | Removes empty/invalid tweets       | Merge first tweet and others | No Operation, do nothing          |                                                                                                                     |
| No Operation, do nothing     | No Operation                 | Ends empty filter branch           | Filter empty ones            |                                  |                                                                                                                     |
| Send error message           | No Operation                 | Placeholder for error notification | Check login                  |                                  |                                                                                                                     |
| REMOVE THIS NODE             | Twitter                      | Deprecated or placeholder node     | DONE                        |                                  |                                                                                                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node:**  
   - Name: `When clicking ‘Test workflow’`  
   - Purpose: Entry to test manually.  
   - Parameters: None; pre-fill `language` and `tweet_url` in pinned data for testing.

2. **Create Code Node for Config:**  
   - Name: `config`  
   - Purpose: Initialize and format input parameters for downstream use.  
   - Connect from manual trigger.

3. **Create Execute Workflow Node:**  
   - Name: `Execute Workflow`  
   - Purpose: Calls main processing sub-workflow.  
   - Connect from `config`.

4. **Create Code Node to Extract Tweets List:**  
   - Name: `Extract tweets list`  
   - Purpose: Parse and prepare list of tweets from input data.  
   - Connect from `Execute Workflow`.

5. **Create Split In Batches Node:**  
   - Name: `MAIN_LOOP`  
   - Purpose: Process tweets one-by-one in batches.  
   - Connect from `Extract tweets list`.  
   - Configure batch size = 1.

6. **Create Langchain LM Chat OpenAI Node:**  
   - Name: `OpenAI Chat Model`  
   - Purpose: Connect to OpenAI API with chat model.  
   - Configure credentials for OpenAI.  
   - Connect as AI model for translator and rewriter agents.

7. **Create Langchain Agent Node for Translation:**  
   - Name: `Translator_openai`  
   - Purpose: Translate tweets using OpenAI chat model.  
   - Connect input from `MAIN_LOOP`.  
   - Connect AI model to `OpenAI Chat Model`.

8. **Create Langchain Agent Node for Rewriting:**  
   - Name: `Rewriter`  
   - Purpose: Rewrites translated tweets for clarity/style.  
   - Connect input from `Translator_openai`.  
   - Connect AI model to `OpenAI Chat Model`.

9. **Create Notion Node to Get Twitter Session:**  
   - Name: `Get twitter session`  
   - Purpose: Retrieve session data from Notion.  
   - Configure Notion credentials and database.  
   - Connect input from `Rewriter`.

10. **Create If Node to Check Session Data:**  
    - Name: `Check session data`  
    - Purpose: Determine if session data exists to proceed or login.  
    - Connect from `Get twitter session`.

11. **Create HTTP Request Node for Twitter Login:**  
    - Name: `Login Twitter`  
    - Purpose: Send login request with credentials.  
    - Configure HTTP method, URL and authentication as per Twitter API.  
    - Connect from `Check session data` (fallback).

12. **Create If Node to Check Login Status:**  
    - Name: `Check login status`  
    - Purpose: Detect if login successful or 2FA required.  
    - Connect from `Login Twitter`.

13. **Create If Node for 2FA Error Handling:**  
    - Name: `login_2fa error`  
    - Purpose: Branch for 2FA requirement.  
    - Connect from `Check login status`.

14. **Create Wait Node for 2FA Code:**  
    - Name: `Wait for 2fa code`  
    - Purpose: Wait webhook for 2FA input.  
    - Configure webhook ID.  
    - Connect from `login_2fa error`.

15. **Create HTTP Request Node for 2FA Login:**  
    - Name: `Login Twitter with 2fa`  
    - Purpose: Complete login with 2FA code.  
    - Connect from `Wait for 2fa code` and fallback from `login_2fa error`.

16. **Create If Node to Check Login Post-2FA:**  
    - Name: `Check login`  
    - Purpose: Confirm login success after 2FA.  
    - Connect from `Login Twitter with 2fa`.

17. **Create Notion Node to Update Twitter Session:**  
    - Name: `Update twitter session`  
    - Purpose: Save updated session info.  
    - Connect from `Check login`.

18. **Create If Node for Authorization Errors:**  
    - Names: `Authorization Error` and `Authorization Error1`  
    - Purpose: Detect authorization failure during posting or login.  
    - Connect from posting check nodes.

19. **Create Set Nodes for Tweet Info:**  
    - Names: `Set tweet infos` and `Set first tweet id and session`  
    - Purpose: Prepare tweet data and store first tweet ID.  
    - Connect from session check and thread identification nodes.

20. **Create HTTP Request Nodes to Post Tweets:**  
    - Names: `Post first tweet` and `Post reply tweet`  
    - Purpose: Post tweets to Twitter API.  
    - Configure endpoints for tweeting and replying.  
    - Connect from respective Set nodes.

21. **Create If Nodes to Check Posting Success:**  
    - Names: `Check posting` and `Check posting1`  
    - Purpose: Verify tweet posting success, branch on failure.  
    - Connect from respective post tweet nodes.

22. **Create Code Node to Manage Iteration Output:**  
    - Name: `set_iteration_outputs`  
    - Purpose: Manage loop state for batch processing.  
    - Connect from `Check posting` and `Check posting1`.

23. **Create Function and Merge Nodes to Extract and Combine Tweet Data:**  
    - Names: `Extract Tweet ID and Username`, `Extract Conversation and Author ID`, `Merge all tweet infos`, `Merge first tweet and others`  
    - Purpose: Parse and aggregate tweet data for thread reconstruction.  
    - Connect in logical sequence following tweet retrieval.

24. **Create HTTP Request Nodes to Fetch Tweets and Replies:**  
    - Names: `Get first tweet`, `Get Tweet Replies`  
    - Purpose: Access Twitter API to retrieve tweets and replies.  
    - Connect appropriately in tweet extraction flow.

25. **Create Filter Node:**  
    - Name: `Filter empty ones`  
    - Purpose: Remove empty or invalid tweets from thread.  
    - Connect after merging tweets.

26. **Create No Operation Nodes for Workflow Control:**  
    - Names: `DONE`, `No Operation, do nothing`, `Send error message`  
    - Purpose: Mark endpoints and placeholders for error handling.

27. **Credential Setup:**  
    - OpenAI API credentials for Langchain nodes.  
    - Twitter OAuth2 or API credentials for HTTP Request nodes related to Twitter.  
    - Notion API credentials for session storage nodes.

28. **Testing & Validation:**  
    - Use manual trigger with example tweet URL and target language.  
    - Validate Twitter API responses, OpenAI outputs, and session management.

---

### 5. General Notes & Resources

| Note Content                                                                                  | Context or Link                                     |
|-----------------------------------------------------------------------------------------------|----------------------------------------------------|
| For Twitter API authentication, ensure OAuth2 credentials are configured with required scopes.| Twitter Developer Portal                            |
| OpenAI API usage requires valid API keys with sufficient quota for translation and rewriting. | https://platform.openai.com/account/api-keys       |
| Notion integration stores session data; configure database and access permissions accordingly.| https://developers.notion.com/                       |
| Webhook configuration for 2FA wait node must be secured to prevent unauthorized access.       | n8n Webhook Security Best Practices                 |
| The workflow includes error handling for authorization failures and retries login accordingly.| Internal workflow logic                              |
| Sticky notes present in the original workflow may contain additional instructions or reminders.| Visible in original n8n editor                       |

---

**Disclaimer:**  
The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. The processing strictly respects applicable content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.