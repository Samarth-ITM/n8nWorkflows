Instagram Content & DM Automation with Gemini, Telegram, and Apify

https://n8nworkflows.xyz/workflows/instagram-content---dm-automation-with-gemini--telegram--and-apify-6070


# Instagram Content & DM Automation with Gemini, Telegram, and Apify

### 1. Workflow Overview

This workflow, named **"Ai Instagram Influencer"**, is a sophisticated Instagram automation suite combining AI-driven Instagram direct message (DM) management, automated user follow actions, and AI-powered Instagram content generation and publishing. It integrates multiple platforms and APIs including Telegram (for chat interface), OpenRouter’s AI models (for natural language processing and content generation), Apify actors (for Instagram interaction automation), Google Sheets (for data storage), and Replicate (for AI image generation).

The workflow is logically divided into three main functional blocks:

- **1.1 Instagram Direct Message (DM) Management via Telegram Chatbot:**  
  Listens for Telegram messages representing Instagram DMs, uses AI to determine intent, performs relevant Instagram actions or replies, and sends responses back through Telegram.

- **1.2 Automated Instagram User Following:**  
  Scheduled hourly execution to read a user list from Google Sheets, automatically follow those users on Instagram, and update the sheet to mark followed users.

- **1.3 AI-Powered Instagram Content Generation and Publishing:**  
  Monthly scheduled generation of 30/31 Instagram post ideas based on a detailed Instagram personality profile, refinement of image prompts and captions, image generation via Replicate, and posting to Instagram through Apify with daily scheduling.

---

### 2. Block-by-Block Analysis

---

#### Block 1.1: Instagram DM Management via Telegram Chatbot

**Overview:**  
This block handles incoming Telegram messages (representing Instagram DMs), identifies the sender and message content, invokes an AI agent to classify the intent (chat reply, running Instagram-related agents, fetching profile info), routes accordingly, and sends Telegram responses back to the user.

**Nodes Involved:**  
- Telegram New Message Trigger  
- Variables1 (set OpenRouter API key, Instagram personality JSON, Instagram session ID)  
- Switch (filters messages by username and verifies presence of message text)  
- Edit Fields (extracts message text)  
- AI Agent (LangChain agent using OpenRouter AI model for intent classification)  
- Structured Output Parser (parses AI JSON output)  
- Switch1 (routes based on AI agent’s action output)  
- Send a text message1 (Telegram reply for chat_back action)  
- run AI influencer actor custom run (calls Apify for custom run_agent action)  
- run AI influencer actor get instagram profile (calls Apify for profile retrieval)  
- Send a text message (Telegram reply for run_agent confirmation)  
- Send a text message2 (Telegram reply for get_instagram_profile confirmation)

**Node Details:**

- **Telegram New Message Trigger:**  
  - Type: webhook trigger for Telegram messages  
  - Credentials: Telegram bot API token  
  - Listens to "message" updates  
  - Input: Telegram user messages  
  - Output: JSON with message data including username and text  
  - Edge cases: Telegram webhook failures, message without text, user blocking bot

- **Variables1:**  
  - Type: Code node  
  - Sets API keys and Instagram personality JSON to workflow context  
  - Injects Instagram session ID and OpenRouter API key for downstream use  
  - Key expressions: hardcoded placeholders to be replaced by user  
  - Edge cases: missing or invalid API keys/session ID causing downstream failures

- **Switch:**  
  - Filters incoming messages for username equals "Wolf23000" or messages containing text  
  - Routes messages accordingly: either to process or ignore  
  - Edge cases: unexpected empty messages, username mismatch leading to no processing

- **Edit Fields:**  
  - Extracts message text into a dedicated JSON field for clarity  
  - Simple field assignment expression

- **AI Agent:**  
  - LangChain agent node using OpenRouter chat model "google/gemini-2.5-flash-preview"  
  - Uses a system prompt instructing to classify message intent into a JSON schema with `action`, `inputMsg`, and `outputMsg` fields  
  - Possible actions: "chat_back", "run_agent", "post_instagram_post", "get_instagram_profile", "follow_instagram_users"  
  - Output parsed by Structured Output Parser  
  - Edge cases: AI timeouts, malformed outputs, API rate limits

- **Structured Output Parser:**  
  - Parses AI Agent JSON string output into structured JSON  
  - Matches schema defined in AI Agent’s system prompt

- **Switch1:**  
  - Routes based on parsed `action` field from AI output JSON  
  - Routes:  
    - "chat_back" → Send a text message1  
    - "run_agent" → run AI influencer actor custom run (Apify)  
    - "get_instagram_profile" → run AI influencer actor get instagram profile (Apify)  
  - Edge cases: unknown action values, routing errors

- **Send a text message1 / Send a text message / Send a text message2:**  
  - Telegram nodes sending messages back to the user  
  - Inputs: text from AI output or Apify call results  
  - Credentials: Telegram API  
  - Edge cases: Telegram API errors, chat ID missing or invalid

- **run AI influencer actor custom run:**  
  - HTTP Request node calling Apify actor API for custom agent run  
  - Passes action "run_agent", AI input message, user personality, Instagram session, OpenRouter API key  
  - Edge cases: API failures, token expiration, malformed request

- **run AI influencer actor get instagram profile:**  
  - Similar to above but action "get_instagram_profile"  
  - Same edge cases

---

#### Block 1.2: Automated Instagram User Following

**Overview:**  
Runs hourly to read a list of Instagram usernames from a Google Sheet, triggers Apify actor to follow these users, and marks them as followed in the sheet.

**Nodes Involved:**  
- Schedule Trigger (hourly)  
- Variables (set API keys, personality JSON, session ID)  
- Variables2 (same as Variables but for this schedule)  
- Get Users to follow (Google Sheets node filtering unfollowed users)  
- Code (prepare usernames array)  
- fllow users with apify actor (HTTP request to Apify to follow users)  
- Code1 (process followed usernames list)  
- Update row in sheet to mark followed users (Google Sheets update)

**Node Details:**

- **Schedule Trigger:**  
  - Fires hourly  
  - Initiates follow user flow

- **Variables / Variables2:**  
  - Code nodes setting API keys etc. (same as in DM block)

- **Get Users to follow:**  
  - Google Sheets node reading users where "followed" column is empty or false  
  - Credentials: Google Sheets OAuth2  
  - Input: none  
  - Output: list of usernames to follow  
  - Edge cases: API limits, empty list, sheet access denied

- **Code:**  
  - Extracts and formats usernames array for Apify call  
  - Input: list of usernames from sheet  
  - Output: JSON with usernames array

- **fllow users with apify actor:**  
  - HTTP Request node calling Apify actor to follow Instagram users  
  - Passes usernames, user personality, session ID, OpenRouter API key  
  - Edge cases: Instagram blocking follow actions, API errors

- **Code1:**  
  - Processes response and prepares data to update sheet marking users as followed

- **Update row in sheet to mark followed users:**  
  - Updates "followed" status to TRUE in Google Sheet  
  - Edge cases: update conflicts, API errors

---

#### Block 1.3: AI-Powered Instagram Content Generation & Publishing

**Overview:**  
Scheduled monthly to generate 30 or 31 Instagram post ideas based on a personality profile, then daily refines image prompts and captions, generates images, updates Google Sheets and posts to Instagram via Apify.

**Nodes Involved:**  
- Schedule Trigger2 (monthly)  
- Variables4 (set personality JSON etc.)  
- AI Agent1 (LangChain agent generating 31 post ideas JSON array)  
- OpenRouter Chat Model (used by AI Agent1)  
- Code2 (parse AI JSON output array)  
- Append the 30 days post generated to sheet (Google Sheets append or update)  
- Schedule Trigger3 (daily at 9 AM)  
- Variables3 (set API keys, personality)  
- Date Formatter (get current date)  
- get today post details (Google Sheets filter by current date)  
- AI Agent2 (refines image prompts and captions)  
- OpenRouter Chat Model2 (used by AI Agent2)  
- get the right values from output (parse AI output)  
- add enhanced prompts to sheet (Google Sheets append or update)  
- generate image with replicate (calls Replicate API to generate image from enhanced prompt)  
- Update row in sheet1 (Google Sheets update with image URL and enhanced prompt fields)  
- Get row(s) in sheet2 (Google Sheets get row(s) by image_url)  
- post to instagram with apify (calls Apify actor to post generated content to Instagram)

**Node Details:**

- **Schedule Trigger2:**  
  - Fires monthly to start post ideas generation

- **Variables4:**  
  - Code node sets personality JSON for content generation

- **AI Agent1:**  
  - LangChain agent with system prompt for generating exactly 31 Instagram post ideas  
  - Each with fields: imagePrompt, caption, scheduledDate  
  - Output: JSON array string

- **OpenRouter Chat Model:**  
  - AI language model (google/gemini-2.5-flash-preview) used by AI Agent1

- **Code2:**  
  - Parses AI JSON output string into JSON array for further processing

- **Append the 30 days post generated to sheet:**  
  - Google Sheets append or update operation for posts generation plan sheet

- **Schedule Trigger3:**  
  - Fires daily at 9 AM for refining and posting

- **Variables3:**  
  - Sets API keys, personality JSON for daily refinement

- **Date Formatter:**  
  - Outputs string of current date in ISO format (YYYY-MM-DD)

- **get today post details:**  
  - Filters Google Sheet rows by current `scheduledDate`

- **AI Agent2:**  
  - LangChain agent refining each post idea: enhancing image prompt and polishing caption  
  - Output: JSON object with `row_number`, `enhancedImagePrompt`, `enhancedCaption`, `scheduledDate`

- **OpenRouter Chat Model2:**  
  - AI language model used by AI Agent2

- **get the right values from output:**  
  - Parses AI agent output JSON string to JSON object

- **add enhanced prompts to sheet:**  
  - Updates Google Sheet with enhanced captions and image prompts

- **generate image with replicate:**  
  - HTTP Request to Replicate API to generate an image from enhanced prompt  
  - Authorization header with Replicate token  
  - Input params: prompt, aspect ratio 9:16, safety filter medium and above, jpg output  
  - Edge cases: API rate limits, failure to generate image

- **Update row in sheet1:**  
  - Updates Google Sheet row with generated image URL

- **Get row(s) in sheet2:**  
  - Reads from Google Sheet by image_url for final post details

- **post to instagram with apify:**  
  - Calls Apify actor to post Instagram image with caption  
  - Uses Instagram session ID and OpenRouter API key  
  - Edge cases: Instagram posting failures, API errors, session expiration

---

### 3. Summary Table

| Node Name                               | Node Type                          | Functional Role                                  | Input Node(s)                      | Output Node(s)                              | Sticky Note                                                                                         |
|----------------------------------------|----------------------------------|-------------------------------------------------|----------------------------------|---------------------------------------------|---------------------------------------------------------------------------------------------------|
| Schedule Trigger                       | Schedule Trigger                 | Hourly trigger for Instagram user follow block  |                                  | Variables                                   |                                                                                                   |
| Variables                             | Code                            | Sets API keys, personality JSON, Instagram session ID for user follow block | Schedule Trigger                 | monitor instagram DM and notifications with apify |                                                                                                   |
| monitor instagram DM and notificatios with apify | HTTP Request                   | Calls Apify to monitor Instagram DMs             | Variables                        |                                  |                                                                                                   |
| Telegram New Message Trigger           | Telegram Trigger                | Listens for Telegram messages (Instagram DM proxy) |                                  | Variables1                                  |                                                                                                   |
| Variables1                           | Code                            | Sets API keys, personality JSON, Instagram session ID for DM block | Telegram New Message Trigger    | Switch                                      |                                                                                                   |
| Switch                               | Switch                         | Filters messages by username and text presence   | Variables1                      | Edit Fields                                  |                                                                                                   |
| Edit Fields                         | Set                            | Extracts message.text field                       | Switch                         | AI Agent                                     |                                                                                                   |
| AI Agent                             | LangChain Agent                | Classifies message intent, outputs structured JSON | Edit Fields                    | Structured Output Parser                      |                                                                                                   |
| Structured Output Parser             | Output Parser                  | Parses AI JSON output                             | AI Agent                      | Switch1                                      |                                                                                                   |
| Switch1                             | Switch                         | Routes based on AI action field                  | Structured Output Parser       | Send a text message1, run AI infulancer actor custom run, run AI infulancer actor get instagram profile |                                                                                                   |
| Send a text message1                 | Telegram                       | Sends chat back response to Telegram user       | Switch1                       |                                             |                                                                                                   |
| run AI infulancer actor custom run   | HTTP Request                   | Calls Apify actor for custom run_agent action    | Switch1                       | Send a text message                           |                                                                                                   |
| run AI infulancer actor get instagram profile | HTTP Request                   | Calls Apify actor for get_instagram_profile action | Switch1                       | Send a text message2                          |                                                                                                   |
| Send a text message                 | Telegram                       | Sends confirmation message for run_agent         | run AI infulancer actor custom run |                                             |                                                                                                   |
| Send a text message2                | Telegram                       | Sends confirmation message for get_instagram_profile | run AI infulancer actor get instagram profile |                                             |                                                                                                   |
| Schedule Trigger1                   | Schedule Trigger               | Hourly trigger for Instagram follow users block |                                  | Variables2                                  |                                                                                                   |
| Variables2                        | Code                          | Sets API keys, personality JSON, Instagram session ID for follow users block | Schedule Trigger1              | Get Users to follow                           |                                                                                                   |
| Get Users to follow                | Google Sheets                 | Retrieves list of users to follow from sheet    | Variables2                    | Code                                         |                                                                                                   |
| Code                             | Code                          | Prepares usernames array for Apify follow action | Get Users to follow            | fllow users with apify actor                  |                                                                                                   |
| fllow users with apify actor       | HTTP Request                   | Calls Apify actor to follow Instagram users      | Code                          | Code1                                        |                                                                                                   |
| Code1                            | Code                          | Prepares data to update followed status          | fllow users with apify actor   | Update row in sheet to mark followed users   |                                                                                                   |
| Update row in sheet to mark followed users | Google Sheets                 | Marks users as followed in Google Sheet          | Code1                         |                                             |                                                                                                   |
| Schedule Trigger2                   | Schedule Trigger               | Monthly trigger for Instagram post generation    |                                  | Variables4                                  |                                                                                                   |
| Variables4                        | Code                          | Sets personality JSON for post generation         | Schedule Trigger2             | AI Agent1                                    |                                                                                                   |
| AI Agent1                        | LangChain Agent              | Generates 31 Instagram post ideas JSON array      | Variables4                   | Code2                                        |                                                                                                   |
| OpenRouter Chat Model             | LangChain LM Chat             | AI model used by AI Agent1                         |                              | AI Agent1                                    |                                                                                                   |
| Code2                            | Code                          | Parses AI JSON output array                        | AI Agent1                    | Append the 30 days post generated to sheet  |                                                                                                   |
| Append the 30 days post generated to sheet | Google Sheets                 | Appends or updates Instagram post ideas in sheet | Code2                        |                                             |                                                                                                   |
| Schedule Trigger3                   | Schedule Trigger               | Daily trigger at 9 AM for post refinement and posting |                              | Variables3                                  |                                                                                                   |
| Variables3                        | Code                          | Sets API keys and personality JSON for daily refinement | Schedule Trigger3            | Date Formatter                                |                                                                                                   |
| Date Formatter                   | Code                          | Outputs current date in YYYY-MM-DD                 | Variables3                   | get today post details                        |                                                                                                   |
| get today post details            | Google Sheets                 | Retrieves posts scheduled for current date        | Date Formatter               | AI Agent2                                    |                                                                                                   |
| AI Agent2                        | LangChain Agent              | Refines image prompts and captions for posts      | get today post details        | get the right values from output              |                                                                                                   |
| OpenRouter Chat Model2            | LangChain LM Chat             | AI model used by AI Agent2                         |                              | AI Agent2                                    |                                                                                                   |
| get the right values from output   | Code                          | Parses AI output JSON string                        | AI Agent2                    | add enhanced prompts to sheet                 |                                                                                                   |
| add enhanced prompts to sheet      | Google Sheets                 | Appends or updates enhanced prompts and captions in sheet | get the right values from output | generate image with replicate                |                                                                                                   |
| generate image with replicate      | HTTP Request                   | Calls Replicate API to generate image from prompt | add enhanced prompts to sheet | Update row in sheet1                          |                                                                                                   |
| Update row in sheet1               | Google Sheets                 | Updates sheet with generated image URL             | generate image with replicate | Get row(s) in sheet2                          |                                                                                                   |
| Get row(s) in sheet2              | Google Sheets                 | Retrieves post data by image_url                    | Update row in sheet1          | post to instagram with apify                  |                                                                                                   |
| post to instagram with apify      | HTTP Request                   | Calls Apify to post Instagram content               | Get row(s) in sheet2          |                                             |                                                                                                   |
| Schedule Trigger                  | Schedule Trigger             | Hourly trigger for Instagram follow block          |                              | Variables                                    |                                                                                                   |
| Sticky Note (multiple)            | Sticky Note                 | Documentation and section labels                    |                              |                                             | See content in Sticky Notes sections below                                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Bot and Credentials:**  
   - Create a bot via BotFather on Telegram, obtain API token.  
   - In n8n, create Telegram API credentials using this token.

2. **Create OpenRouter API Credentials:**  
   - Register at OpenRouter, generate API key.  
   - Add API key as credential in n8n for OpenRouter API.

3. **Prepare Instagram Session ID:**  
   - Extract valid Instagram session ID cookie from your browser (note the risk).  
   - Store as a workflow variable or environment variable for use in Code nodes.

4. **Create Google Sheets Credentials:**  
   - Setup Google Cloud project with Google Sheets API enabled.  
   - Configure OAuth2 credentials in n8n.  
   - Prepare two Google Sheets:  
     - "Usernames to follow" sheet with columns: usernames, followed, row_number  
     - "posts generation plan" sheet with columns: imagePrompt, caption, scheduledDate, enhancedImagePrompt, enhancedCaption, image_url, row_number

5. **Set Up Telegram New Message Trigger Node:**  
   - Node type: Telegram Trigger  
   - Set update type to "message"  
   - Use created Telegram credentials.

6. **Variables1 Code Node:**  
   - Add JavaScript code to inject OpenRouter API key, Instagram personality JSON (as shown in workflow), and Instagram session ID into each incoming message item.  
   - Connect Telegram Trigger → Variables1.

7. **Switch Node:**  
   - Configure to filter messages where username equals "Wolf23000" or messages with text exist.  
   - Connect Variables1 → Switch.

8. **Edit Fields Node:**  
   - Extract message text into a dedicated field `message.text`.  
   - Connect Switch → Edit Fields.

9. **AI Agent Node:**  
   - Configure LangChain Agent node with OpenRouter Chat Model (`google/gemini-2.5-flash-preview`).  
   - Use system prompt to classify message intent into JSON schema as per workflow.  
   - Connect Edit Fields → AI Agent.

10. **Structured Output Parser Node:**  
    - Configure to parse the JSON output schema from AI Agent.  
    - Connect AI Agent → Structured Output Parser.

11. **Switch1 Node:**  
    - Route based on parsed `action` field:  
        - `chat_back` → Send a text message1  
        - `run_agent` → HTTP Request to Apify custom run  
        - `get_instagram_profile` → HTTP Request to Apify get profile  
    - Connect Structured Output Parser → Switch1.

12. **Send a text message1 Node:**  
    - Telegram node to send AI chat response back to Telegram user.  
    - Connect Switch1 (chat_back branch) → Send a text message1.

13. **run AI influencer actor custom run Node:**  
    - HTTP Request node calling Apify API with `run_agent` action, passing relevant data.  
    - Connect Switch1 (run_agent branch) → run AI influencer actor custom run.

14. **Send a text message Node:**  
    - Telegram node to confirm run_agent response.  
    - Connect run AI influencer actor custom run → Send a text message.

15. **run AI influencer actor get instagram profile Node:**  
    - HTTP Request node calling Apify API with `get_instagram_profile` action.  
    - Connect Switch1 (get_instagram_profile branch) → run AI influencer actor get instagram profile.

16. **Send a text message2 Node:**  
    - Telegram node to confirm profile info response.  
    - Connect run AI influencer actor get instagram profile → Send a text message2.

17. **Schedule Trigger Node (Hourly):**  
    - Set to trigger every hour for user following automation.  
    - Connect → Variables (code node same as Variables1 but for follow block).

18. **Variables2 Code Node:**  
    - Same setup as Variables1, inject API keys and session info.  
    - Connect Schedule Trigger1 → Variables2.

19. **Get Users to follow Node:**  
    - Google Sheets node to read usernames where "followed" is empty or false.  
    - Connect Variables2 → Get Users to follow.

20. **Code Node:**  
    - Extract usernames array for Apify call.  
    - Connect Get Users to follow → Code.

21. **fllow users with apify actor Node:**  
    - HTTP Request node calling Apify actor to follow Instagram users.  
    - Connect Code → fllow users with apify actor.

22. **Code1 Node:**  
    - Prepare list of followed users for sheet update.  
    - Connect fllow users with apify actor → Code1.

23. **Update row in sheet to mark followed users Node:**  
    - Update Google Sheet rows marking users as followed.  
    - Connect Code1 → Update row in sheet to mark followed users.

24. **Schedule Trigger2 Node (Monthly):**  
    - Set to trigger monthly for post idea generation.  
    - Connect → Variables4 Code Node.

25. **Variables4 Code Node:**  
    - Inject personality JSON for post generation.  
    - Connect Schedule Trigger2 → Variables4.

26. **AI Agent1 Node:**  
    - LangChain agent to generate 31 post ideas JSON array.  
    - Connect Variables4 → AI Agent1.

27. **OpenRouter Chat Model Node:**  
    - Configure with OpenRouter model (gemini-2.5-flash-preview).  
    - Connect AI Agent1 → OpenRouter Chat Model.

28. **Code2 Node:**  
    - Parse AI JSON output array for posts.  
    - Connect AI Agent1 → Code2.

29. **Append the 30 days post generated to sheet Node:**  
    - Append or update Google Sheet "posts generation plan" with generated posts.  
    - Connect Code2 → Append the 30 days post generated to sheet.

30. **Schedule Trigger3 Node (Daily at 9 AM):**  
    - Trigger daily refinement and auto-posting.  
    - Connect → Variables3 Code Node.

31. **Variables3 Code Node:**  
    - Inject API keys and personality JSON for refinement.  
    - Connect Schedule Trigger3 → Variables3.

32. **Date Formatter Node:**  
    - Output current date string.  
    - Connect Variables3 → Date Formatter.

33. **get today post details Node:**  
    - Google Sheets node to get posts scheduled for today's date.  
    - Connect Date Formatter → get today post details.

34. **AI Agent2 Node:**  
    - Refine image prompts and captions for posts.  
    - Connect get today post details → AI Agent2.

35. **OpenRouter Chat Model2 Node:**  
    - OpenRouter model for AI Agent2.  
    - Connect AI Agent2 → OpenRouter Chat Model2.

36. **get the right values from output Node:**  
    - Parse AI output string to JSON.  
    - Connect AI Agent2 → get the right values from output.

37. **add enhanced prompts to sheet Node:**  
    - Update Google Sheet with enhanced captions and prompts.  
    - Connect get the right values from output → add enhanced prompts to sheet.

38. **generate image with replicate Node:**  
    - HTTP Request to Replicate API for AI image generation.  
    - Connect add enhanced prompts to sheet → generate image with replicate.

39. **Update row in sheet1 Node:**  
    - Update Google Sheet with generated image URL.  
    - Connect generate image with replicate → Update row in sheet1.

40. **Get row(s) in sheet2 Node:**  
    - Retrieve post details by image_url from sheet.  
    - Connect Update row in sheet1 → Get row(s) in sheet2.

41. **post to instagram with apify Node:**  
    - HTTP Request to Apify actor to post Instagram content.  
    - Connect Get row(s) in sheet2 → post to instagram with apify.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                | Context or Link                                                                                  |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Workflow integrates Telegram chatbot to handle Instagram DMs, OpenRouter AI for intelligent message classification and content generation, Apify actors for Instagram automation, Google Sheets for data management, and Replicate for AI image generation.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      | Overview of workflow components                                                                  |
| Instagram session ID usage requires caution due to Instagram's ToS restricting automation; risk of account suspension.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               | Instagram session ID handling warning                                                            |
| OpenRouter API key must be kept secret; monitor usage to avoid unexpected costs.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           | OpenRouter API credential management                                                             |
| Google Sheets API quotas are generous but watch for limits if scaling.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | Google Sheets API notes                                                                          |
| Telegram API is free and reliable; ensure bot permissions are correct.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | Telegram API usage                                                                              |
| Replicate API key included in HTTP header; usage may incur costs depending on model usage.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 | Replicate API integration                                                                        |
| Apify actors used for Instagram interactions abstract away direct Instagram API calls, improving reliability but requiring valid tokens and session IDs.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            | Apify actor usage                                                                                |
| Instagram personality JSON template embedded in code nodes for AI content customization; customize before running.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | Personality JSON template                                                                        |
| Workflow uses LangChain integration in n8n for AI agent orchestration with OpenRouter models.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             | LangChain OpenRouter integration                                                                 |
| Sticky notes provide internal documentation and section labels, including detailed workflow architecture and setup instructions.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | Workflow internal documentation                                                                 |
| Recommended n8n version as of July 2025: v1.101.1 or later. System requirements: Node.js 16+, 2+ cores, 2+ GB RAM, PostgreSQL recommended for production.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               | System and version requirements                                                                 |
| Contact info for author: Mohamed el Hadi Msaid, email: mohamedgb00714@gmail.com, LinkedIn: https://www.linkedin.com/in/mohamed-el-hadi-m-said-1a9a5095/                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  | Author contact                                                                                   |

---

This document comprehensively details the AI Instagram Influencer n8n workflow for advanced users and AI agents, enabling understanding, reproduction, and maintenance.