Generate AI Images in Telegram with GPT-4o Enhancement and Flux Pro

https://n8nworkflows.xyz/workflows/generate-ai-images-in-telegram-with-gpt-4o-enhancement-and-flux-pro-6651


# Generate AI Images in Telegram with GPT-4o Enhancement and Flux Pro

### 1. Workflow Overview

This workflow implements an AI-powered image generation Telegram bot enhanced with GPT-4o prompt refinement and Flux Pro AI/ML image generation API. It targets users who want to create AI-generated images by sending text prompts to the Telegram bot. The workflow includes usage tracking with daily request limits enforced via Google Sheets, prompt enhancement for better image generation quality, image creation through an advanced AI model, vivid image description generation, delivery of images with captions back to Telegram users, and logging all generated images with metadata.

Logical blocks:

- **1.1 Input Reception:** Receive and process incoming Telegram messages.
- **1.2 Usage Limit Check:** Retrieve user usage logs from Google Sheets, count today's requests, compare against a daily limit, and notify if exceeded.
- **1.3 Prompt Enhancement:** Use GPT-4o to enrich and structure the user prompt for improved image generation.
- **1.4 Image Generation:** Generate an image using the enhanced prompt with Flux Pro model via AIMLAPI.
- **1.5 Image Description:** Use GPT-4o to create a vivid, sensory description of the generated image.
- **1.6 Delivery & Logging:** Send the image and description to the user via Telegram and log the generation details to Google Sheets.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block listens for new messages sent to the Telegram bot and triggers the workflow when a user sends a message.

- **Nodes Involved:**  
  - 📩 Receive Telegram Message

- **Node Details:**

  - **📩 Receive Telegram Message**  
    - Type: Telegram Trigger  
    - Role: Entry point; listens for new Telegram messages (update type: "message").  
    - Configuration: Monitors only message updates, no additional filters.  
    - Inputs: N/A (trigger node)  
    - Outputs: Passes message JSON data downstream  
    - Version: 1.2  
    - Edge Cases: Telegram webhook failures, malformed messages, user blocking the bot, connection timeouts.  
    - Credentials: Telegram API OAuth configured.

---

#### 1.2 Usage Limit Check

- **Overview:**  
  This block fetches the user's image generation usage from Google Sheets, counts how many requests have been made today, sets a daily limit, checks if the limit is exceeded, and notifies the user if so.

- **Nodes Involved:**  
  - 📊 Fetch Usage Logs  
  - 📈 Count Today’s Requests  
  - 🔢 Set Daily Limit  
  - 🚦 Check Limit Exceeded?  
  - 🚫 Notify: Limit Exceeded

- **Node Details:**

  - **📊 Fetch Usage Logs**  
    - Type: Google Sheets  
    - Role: Retrieves usage entries matching the current user ID and today's date.  
    - Configuration: Filters rows where `user_id` matches the Telegram sender ID and `date` matches current date (ISO string sliced to YYYY-MM-DD).  
    - Inputs: Telegram message JSON from previous node  
    - Outputs: Filtered list of usage logs for counting  
    - Version: 4.6  
    - Edge Cases: Google Sheets API errors, auth failures, empty or missing data, date format mismatch.  
    - Credentials: Google Sheets OAuth2.

  - **📈 Count Today’s Requests**  
    - Type: Aggregate  
    - Role: Counts the number of usage log entries returned (i.e., number of generations today).  
    - Configuration: Aggregates all item data length from previous node output.  
    - Inputs: Usage logs array  
    - Outputs: Number of requests today  
    - Version: 1  
    - Edge Cases: Empty input arrays, unexpected data structures.

  - **🔢 Set Daily Limit**  
    - Type: Set  
    - Role: Defines a fixed daily limit value for allowed image generations (default: 5).  
    - Configuration: Sets variable `daily_limit = 5`.  
    - Inputs: Number of requests today (aggregated)  
    - Outputs: Passes daily limit downstream  
    - Version: 3.4  
    - Edge Cases: None significant; configurable value.

  - **🚦 Check Limit Exceeded?**  
    - Type: If  
    - Role: Compares the number of requests today against the daily limit to decide workflow path.  
    - Configuration: Condition checks if `count of requests < daily_limit`.  
    - Inputs: Number of requests and daily limit  
    - Outputs:  
      - True branch: Continue processing (limit not exceeded)  
      - False branch: Notify user limit exceeded  
    - Version: 2.2  
    - Edge Cases: Expression evaluation errors, undefined variables.

  - **🚫 Notify: Limit Exceeded**  
    - Type: Telegram  
    - Role: Sends a Telegram message informing the user that their daily limit has been exceeded.  
    - Configuration: Sends a message: "Sorry! Your *daily limit of X generations* is exceeded!" with Markdown formatting to the user's chat ID.  
    - Inputs: From the False branch of limit check, uses daily_limit and chat id expressions.  
    - Outputs: Terminates or ends workflow for this user message.  
    - Version: 1.2  
    - Edge Cases: Telegram API errors, user blocking bot, malformed chat IDs.

---

#### 1.3 Prompt Enhancement

- **Overview:**  
  This block uses GPT-4o to enhance the user's original prompt into a more detailed, expressive version to improve the quality of image generation.

- **Nodes Involved:**  
  - 🧠 Enhance Prompt (AI/ML API | GPT-4o)

- **Node Details:**

  - **🧠 Enhance Prompt (AI/ML API | GPT-4o)**  
    - Type: AIMLAPI (OpenAI GPT-4o)  
    - Role: Receives the raw user text and returns a refined prompt optimized for image generation.  
    - Configuration:  
      - Model: openai/gpt-4o  
      - Prompt: Instructional system prompt specifying to enhance user input into a rich, structured prompt without explanation.  
      - Input: User's raw message text from Telegram node.  
    - Inputs: User message text JSON  
    - Outputs: Enhanced prompt text (`content` field)  
    - Version: 1  
    - Edge Cases: API errors, rate limiting, prompt truncation, malformed input.  
    - Credentials: AI/ML API key for AIMLAPI.

---

#### 1.4 Image Generation

- **Overview:**  
  Generates a single AI image using the enhanced prompt by calling the Flux Pro model on AIMLAPI.

- **Nodes Involved:**  
  - 🎨 Generate Image (AI/ML API | flux-pro)

- **Node Details:**

  - **🎨 Generate Image (AI/ML API | flux-pro)**  
    - Type: HTTP Request  
    - Role: Sends a POST request to AIMLAPI's image generation endpoint with the enhanced prompt.  
    - Configuration:  
      - URL: `https://api.aimlapi.com/v1/images/generations`  
      - Method: POST  
      - Body parameters:  
        - model: flux-pro  
        - prompt: enhanced prompt from previous node (`content`)  
        - n: 1 (generate one image)  
        - size: 1024x1024  
        - response_formats: 1 (presumably URL)  
      - Authentication: Predefined AIMLAPI credentials  
    - Inputs: Enhanced prompt text  
    - Outputs: JSON response containing image URL(s)  
    - Version: 4.2  
    - Edge Cases: API downtime, invalid prompt, network errors, rate limits.

---

#### 1.5 Image Description

- **Overview:**  
  Creates a vivid, sensory description of the generated image to accompany the image when sent to the user.

- **Nodes Involved:**  
  - 🖋 Describe Image (AI/ML API | GPT-4o)

- **Node Details:**

  - **🖋 Describe Image (AI/ML API | GPT-4o)**  
    - Type: AIMLAPI (OpenAI GPT-4o)  
    - Role: Takes the image generation prompt and writes a short, elegant description as if the image already exists.  
    - Configuration:  
      - Model: openai/gpt-4o  
      - Prompt: Template instructing to output 1-2 sentence vivid description starting with "Here’s your image:"  
      - Input: The prompt used for image generation (from image generation node's JSON `prompt`)  
    - Inputs: Prompt text  
    - Outputs: Text description (`content`)  
    - Version: 1  
    - Edge Cases: API failures, prompt parsing errors, incomplete description generation.

---

#### 1.6 Delivery & Logging

- **Overview:**  
  Sends the generated image with its description caption to the Telegram user and logs the generation event with metadata to Google Sheets.

- **Nodes Involved:**  
  - 📤 Send Image to User  
  - 📝 Log Successful Generation

- **Node Details:**

  - **📤 Send Image to User**  
    - Type: Telegram  
    - Role: Sends the generated image as a photo to the user's chat, with the descriptive caption.  
    - Configuration:  
      - Operation: sendPhoto  
      - File: Image URL from image generation response  
      - Caption: Description from previous node  
      - Chat ID: From original Telegram message  
      - Append Attribution: Disabled  
    - Inputs: Image URL and description text  
    - Outputs: Success confirmation  
    - Version: 1.2  
    - Edge Cases: Telegram API errors, invalid URLs, user blocking bot.

  - **📝 Log Successful Generation**  
    - Type: Google Sheets  
    - Role: Appends a new row logging user ID, date (YYYY-MM-DD), original query, and image URL.  
    - Configuration:  
      - Sheet: "Sheet1" in specified Google Sheets document  
      - Columns: user_id, date, query, result_url  
      - Mapping: Extracts data from Telegram message and image generation nodes  
      - Operation: Append  
    - Inputs: Telegram user info and image URL  
    - Outputs: Logging confirmation  
    - Version: 4.6  
    - Edge Cases: Google Sheets API failures, permission issues, rate limits.

---

### 3. Summary Table

| Node Name                          | Node Type                  | Functional Role                                  | Input Node(s)                  | Output Node(s)                 | Sticky Note                                                                                                          |
|-----------------------------------|----------------------------|-------------------------------------------------|-------------------------------|-------------------------------|----------------------------------------------------------------------------------------------------------------------|
| 📩 Receive Telegram Message        | Telegram Trigger           | Receives incoming Telegram messages              | N/A                           | 📊 Fetch Usage Logs            | ## Incoming Message<br>### Handle incoming user messages from Telegram.                                              |
| 📊 Fetch Usage Logs                | Google Sheets              | Retrieves user's usage logs for today             | 📩 Receive Telegram Message   | 📈 Count Today’s Requests      | ##  Usage Limit Check<br>### Check how many times the user has generated today and enforce the daily limit.          |
| 📈 Count Today’s Requests          | Aggregate                  | Counts number of requests logged today            | 📊 Fetch Usage Logs            | 🔢 Set Daily Limit             | ##  Usage Limit Check<br>### Check how many times the user has generated today and enforce the daily limit.          |
| 🔢 Set Daily Limit                 | Set                        | Defines the daily image generation limit          | 📈 Count Today’s Requests      | 🚦 Check Limit Exceeded?       | ##  Usage Limit Check<br>### Check how many times the user has generated today and enforce the daily limit.          |
| 🚦 Check Limit Exceeded?           | If                         | Compares usage count to limit, branches workflow  | 🔢 Set Daily Limit             | 🧠 Enhance Prompt (True branch)<br>🚫 Notify: Limit Exceeded (False branch) | ##  Usage Limit Check<br>### Check how many times the user has generated today and enforce the daily limit.<br>---<br>##  Notify: Limit Exceeded<br>### Notify the user that the daily generation limit has been reached |
| 🚫 Notify: Limit Exceeded          | Telegram                   | Notify user that daily generation limit exceeded | 🚦 Check Limit Exceeded?       | N/A                           | ##  Notify: Limit Exceeded<br>### Notify the user that the daily generation limit has been reached                    |
| 🧠 Enhance Prompt (AI/ML API | GPT-4o) | AIMLAPI (OpenAI GPT-4o)   | Enhances user input prompt for image generation  | 🚦 Check Limit Exceeded? (True branch) | 🎨 Generate Image (AI/ML API | flux-pro)  | ##  Prompt Enhancement<br>### Use LLM to rewrite the user's input into a rich and detailed prompt.                   |
| 🎨 Generate Image (AI/ML API | flux-pro) | HTTP Request               | Generates AI image using enhanced prompt          | 🖋 Describe Image (AI/ML API | GPT-4o) | ##  Image Generation<br>### Generate an image using the enhanced prompt via AIMLAPI.                                |
| 🖋 Describe Image (AI/ML API | GPT-4o) | AIMLAPI (OpenAI GPT-4o)   | Creates vivid description of generated image      | 📤 Send Image to User          | ##  Image Description<br>### Create a vivid description of the generated image using LLM.                            |
| 📤 Send Image to User              | Telegram                   | Sends generated image and caption to user         | 🖋 Describe Image (AI/ML API | GPT-4o) | 📝 Log Successful Generation    | ## Delivery & Logging<br>### Send the final image and caption to the user, and log the result to Google Sheets.       |
| 📝 Log Successful Generation       | Google Sheets              | Logs generation event metadata                      | 📤 Send Image to User          | N/A                           | ## Delivery & Logging<br>### Send the final image and caption to the user, and log the result to Google Sheets.       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node**  
   - Type: Telegram Trigger  
   - Configure to listen for "message" updates.  
   - Connect to your Telegram bot by creating Telegram API credentials with the bot token from @BotFather.  
   - This is the starting node.

2. **Add Google Sheets Node to Fetch Usage Logs**  
   - Type: Google Sheets  
   - Operation: Read with filter  
   - Configure to read from your Google Sheet document with a sheet named "Sheet1".  
   - Filter rows where `user_id` equals `={{ $json.message.from.id }}` and `date` equals `={{ new Date().toISOString().slice(0,10) }}` (today’s date).  
   - Authenticate via Google Sheets OAuth2 credentials.

3. **Add Aggregate Node to Count Requests**  
   - Type: Aggregate  
   - Operation: Aggregate all item data  
   - Count the length of the filtered rows from the previous node.

4. **Add Set Node for Daily Limit**  
   - Type: Set  
   - Add a numeric field named `daily_limit` with a value, e.g., 5.  
   - This sets the allowed daily generation limit.

5. **Add If Node to Check Limit**  
   - Type: If  
   - Condition: Check if the count of today’s requests (from aggregate node) is less than `daily_limit`.  
   - True branch means user is allowed to generate; False means limit exceeded.

6. **Add Telegram Node to Notify Limit Exceeded**  
   - Type: Telegram  
   - Operation: Send Message  
   - Message: "Sorry! Your *daily limit of {{ daily_limit }} generations* is exceeded!"  
   - Send to the chat ID from the incoming Telegram message.  
   - Connect from False branch of the If node.  
   - Use Telegram API credentials.

7. **Add AIMLAPI Node for Prompt Enhancement**  
   - Type: AIMLAPI (OpenAI GPT-4o)  
   - Model: openai/gpt-4o  
   - Prompt: Instruct the model to enhance the user’s raw prompt into a rich and detailed prompt, output only final prompt.  
   - Input: Original message text from Telegram node.  
   - Connect from True branch of the If node.  
   - Use AIMLAPI credentials.

8. **Add HTTP Request Node to Generate Image**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.aimlapi.com/v1/images/generations`  
   - Body (JSON):  
     ```json
     {
       "model": "flux-pro",
       "prompt": "={{ $json.content }}",
       "n": 1,
       "size": "1024x1024",
       "response_formats": 1
     }
     ```  
   - Authentication: Use AIMLAPI API key (predefined credentials).  
   - Connect after prompt enhancement node.

9. **Add AIMLAPI Node for Image Description**  
   - Type: AIMLAPI (OpenAI GPT-4o)  
   - Prompt: Template instructing to write a vivid, sensory description starting with "Here's your image:" based on the prompt.  
   - Input: The prompt used for image generation (from previous node's JSON).  
   - Connect after image generation node.

10. **Add Telegram Node to Send Image**  
    - Type: Telegram  
    - Operation: sendPhoto  
    - File: Use the image URL from the image generation node response.  
    - Caption: Use the description output from the previous step.  
    - Chat ID: Original Telegram chat ID.  
    - Connect after image description node.  
    - Use Telegram API credentials.

11. **Add Google Sheets Node to Log Generation**  
    - Type: Google Sheets  
    - Operation: Append  
    - Sheet: "Sheet1"  
    - Columns to log: `user_id` (Telegram user ID), `date` (today’s date), `query` (original user prompt), `result_url` (image URL).  
    - Connect after sending image node.  
    - Use Google Sheets OAuth2 credentials.

**Connect nodes in the order described:**  
📩 Receive Telegram Message → 📊 Fetch Usage Logs → 📈 Count Today’s Requests → 🔢 Set Daily Limit → 🚦 Check Limit Exceeded?  
- False branch → 🚫 Notify: Limit Exceeded  
- True branch → 🧠 Enhance Prompt → 🎨 Generate Image → 🖋 Describe Image → 📤 Send Image → 📝 Log Successful Generation

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                            | Context or Link                                                                                                          |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------|
| # 🧠 AI Image Generator Bot — Telegram + AIMLAPI (via n8n) This workflow lets users generate AI images by messaging a Telegram bot. Each user has a **daily limit**, tracked using **Google Sheets**. Prompts are enhanced and images are generated using **AIMLAPI**. Features include text-to-image via `flux-pro`, GPT-4o prompt enhancement, descriptive captions, Telegram delivery, usage tracking, and logging. | Overview sticky note in workflow describing the project purpose and features.                                            |
| ## ⚙️ Customization You can adjust the daily limit in the `Set Daily Limit` node. Additional enhancements can include NSFW filtering, dynamic model selection, or supporting commands like `/help` or `/history`. Example user flow is to send a prompt, receive a description and image.                                                                                                     | Sticky note with customization tips and example user flow.                                                               |
| # 🛠 Setup Guide Step 1: Create Telegram Bot via @BotFather and get API token. Step 2: Add Telegram API credentials in n8n. Step 3: Create Google Sheet named "Sheet1" with headers: `user_id | date | query | result_url`. Step 4: Setup credentials for Google Sheets (OAuth2 or service account) and AIMLAPI (API key). Base URL: https://api.aimlapi.com/v1 Docs: https://docs.aimlapi.com                                      | Setup instructions sticky note.                                                                                          |
| ## 📁 Data Logged Each generation is logged with `user_id`, `date`, `query`, and `result_url`. Testing tips: Use a test Telegram chat, inspect payloads with Console or Set nodes, and trigger workflow from Telegram, not n8n’s Execute Node when testing with live data.                                                                                                                 | Notes on logging and testing from sticky note.                                                                           |
| **Notify: Limit Exceeded** node sends formatted Markdown messages to users who exceed their daily generation quota to ensure resource control and user awareness.                                                                                                                                                                                                                      | Sticky note emphasizing the user notification mechanism.                                                                 |

---

This detailed documentation covers all nodes, their purpose, configuration, flow, edge cases, and instructions to reproduce the entire workflow manually in n8n without requiring the JSON export. It also preserves all sticky note content and references links for additional setup and usage guidance.