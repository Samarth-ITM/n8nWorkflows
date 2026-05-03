Send Daily Inspirational Quotes with Gemini Translation to Telegram Subscribers

https://n8nworkflows.xyz/workflows/send-daily-inspirational-quotes-with-gemini-translation-to-telegram-subscribers-7688


# Send Daily Inspirational Quotes with Gemini Translation to Telegram Subscribers

### 1. Workflow Overview

This workflow, named **"Send Daily Inspirational Quotes with Gemini Translation to Telegram Subscribers"**, is designed to automatically send daily inspirational quotes to Telegram users subscribed to the bot. It fetches random quotes from an external API, translates and stylizes them using Google Gemini AI, manages user subscriptions via Google Sheets, and delivers formatted messages to all registered subscribers.

The workflow is logically divided into two main flows:

- **Daily Quote Sending Flow (Scheduled)**
  - Fetches a random inspirational quote daily.
  - Translates and enhances the quote with emoji stickers using Google Gemini AI.
  - Reads the list of registered subscribers from Google Sheets.
  - Sends the formatted quote message to each subscriber on Telegram.

- **User Registration Flow (Trigger-based)**
  - Detects when a user messages the Telegram bot.
  - Registers the user by adding their Telegram user ID to a Google Sheet.
  - Sends a welcome message to the new subscriber.

The workflow’s key logical blocks are:

- 1.1 Daily Schedule Trigger  
- 1.2 Quote Fetching from ZenQuotes API  
- 1.3 AI Processing (Translation + Stickerization)  
- 1.4 Subscriber Management with Google Sheets  
- 1.5 Message Delivery to Telegram Subscribers  
- 2.1 Telegram User Registration Trigger  
- 2.2 Register User in Google Sheets  
- 2.3 Welcome Message to New User

---

### 2. Block-by-Block Analysis

#### 1.1 Daily Schedule Trigger

**Overview:**  
This block triggers the daily quote sending process automatically once per day.

**Nodes Involved:**

- Schedule Trigger
- Set Language

**Node Details:**

- **Schedule Trigger**  
  - Type: Trigger node (Schedule Trigger)  
  - Configuration: Fires once daily (default interval)  
  - Inputs: None (trigger)  
  - Outputs: Triggers `Set Language` node  
  - Edge cases: Possible failures if n8n is offline or scheduler malfunctions; no direct auth issues.

- **Set Language**  
  - Type: Set node  
  - Configuration: Defines the target translation language as Persian (`Translate_lang = "Persian"`)  
  - Inputs: Receives trigger from Schedule Trigger  
  - Outputs: Passes language setting to `HTTP Request` node  
  - Edge cases: If the language is changed or set incorrectly, translation output might be invalid.

---

#### 1.2 Quote Fetching from ZenQuotes API

**Overview:**  
Retrieves a random inspirational quote from the ZenQuotes API.

**Nodes Involved:**

- HTTP Request

**Node Details:**

- **HTTP Request**  
  - Type: HTTP Request node  
  - Configuration:  
    - URL: `https://zenquotes.io/api/random`  
    - Method: GET (default)  
    - No special headers or auth required  
  - Inputs: Receives language setting from `Set Language`  
  - Outputs: Passes quote JSON to `Basic LLM Chain` node  
  - Edge cases: API downtime, rate limiting, or malformed response may cause errors; no auth issues.

---

#### 1.3 AI Processing (Translation + Stickerization)

**Overview:**  
Translates the fetched quote into the target language and adds emoji stickers to both original and translated quotes using Google Gemini AI.

**Nodes Involved:**

- Basic LLM Chain
- Google Gemini Chat Model

**Node Details:**

- **Basic LLM Chain**  
  - Type: Langchain LLM Chain node  
  - Configuration:  
    - Uses a prompt that instructs to translate the quote into the target language and “stickerize” it (add related emojis).  
    - Input text references the quote (`$json.q`) and language from previous nodes.  
    - Expects a response containing both the original and translated “stickerized” quotes without labels.  
  - Inputs: Receives raw quote JSON from `HTTP Request` and language info  
  - Outputs: Passes formatted text to `Google Sheets1`  
  - Edge cases: Incorrect prompt or API failures could result in invalid or missing translations.

- **Google Gemini Chat Model**  
  - Type: Langchain Google Gemini language model node  
  - Configuration:  
    - Model: `models/gemini-2.5-flash`  
    - Credentials: Google PaLM API key  
  - Inputs: Connected as the AI language model backend for the `Basic LLM Chain` node  
  - Outputs: Provides AI-generated translations and stylizations  
  - Edge cases: API errors, quota limits, network timeouts, or authentication failures.

---

#### 1.4 Subscriber Management with Google Sheets

**Overview:**  
Manages the list of Telegram subscribers by reading and writing user IDs in a Google Sheet.

**Nodes Involved:**

- Google Sheets (User Registration)
- Google Sheets1 (User List for Sending)

**Node Details:**

- **Google Sheets (User Registration)**  
  - Type: Google Sheets node  
  - Configuration:  
    - Operation: Append  
    - Sheet: Sheet1 (`gid=0`) in a specific Google Sheet document  
    - Columns: `registered_users` (string, user ID), `date` (timestamp)  
  - Inputs: Receives Telegram user ID from `Telegram Trigger`  
  - Outputs: Passes data to `Telegram` node (welcome message)  
  - Edge cases: API quota limits, credential expiration, or permission issues.

- **Google Sheets1 (User List for Sending)**  
  - Type: Google Sheets node  
  - Configuration:  
    - Operation: Read (default) entire sheet with registered user IDs  
    - Same sheet as above  
  - Inputs: Receives formatted quote from `Basic LLM Chain`  
  - Outputs: Passes list of subscribers for message sending  
  - Edge cases: Empty sheet or read errors could prevent message delivery.

---

#### 1.5 Message Delivery to Telegram Subscribers

**Overview:**  
Sends the translated and stickerized quote to all registered Telegram users in MarkdownV2 format.

**Nodes Involved:**

- Send a text message

**Node Details:**

- **Send a text message**  
  - Type: Telegram node  
  - Configuration:  
    - Text uses MarkdownV2 formatting with escaped periods to avoid Markdown parsing errors.  
    - Message format includes the original quote and translated text with stickers.  
    - `chatId` dynamically set to each registered user from Google Sheets (`$json.registered_users`).  
    - `parse_mode` is set to `MarkdownV2`.  
  - Inputs: Receives user list from `Google Sheets1`  
  - Outputs: None (end node)  
  - Credentials: Telegram Bot Token (`SgsBot`)  
  - Edge cases: User blocked bot, invalid chatId, Telegram API rate limits, malformed Markdown causing message failures.

---

#### 2.1 Telegram User Registration Trigger

**Overview:**  
Detects when a Telegram user messages the bot to register them automatically.

**Nodes Involved:**

- Telegram Trigger
- Google Sheets
- Telegram (Welcome Message)

**Node Details:**

- **Telegram Trigger**  
  - Type: Telegram Trigger node  
  - Configuration: Listens for any Telegram message updates  
  - Inputs: None (trigger)  
  - Outputs: Passes new user message data to `Google Sheets` node  
  - Edge cases: Webhook setup errors, Telegram downtime, or missing permissions.

- **Telegram**  
  - Type: Telegram node  
  - Configuration:  
    - Sends a welcome message: "👋 Welcome, I will send you daily quote."  
    - `chatId` is the sender’s Telegram user ID from the trigger  
  - Inputs: Receives confirmation after user ID is appended to Google Sheets  
  - Outputs: None  
  - Credentials: Telegram Bot Token (`SgsBot`)  
  - Edge cases: User blocking the bot, message send failure, Telegram API issues.

---

### 3. Summary Table

| Node Name          | Node Type                          | Functional Role                 | Input Node(s)           | Output Node(s)        | Sticky Note                                                                                  |
|--------------------|----------------------------------|--------------------------------|-------------------------|-----------------------|----------------------------------------------------------------------------------------------|
| Schedule Trigger    | Schedule Trigger                  | Daily scheduler trigger        | -                       | Set Language           | **DAILY SCHEDULE:** Triggers once per day to send quotes                                     |
| Set Language       | Set                              | Defines target translation language | Schedule Trigger        | HTTP Request           |                                                                                              |
| HTTP Request       | HTTP Request                     | Fetch random quote from API     | Set Language             | Basic LLM Chain        | **QUOTE FETCHING** Gets random quote from ZenQuotes API                                      |
| Basic LLM Chain    | Langchain LLM Chain              | Translate + add emojis          | HTTP Request             | Google Sheets1         | **AI PROCESSING:** Translates + adds emojis with Gemini                                      |
| Google Gemini Chat Model | Langchain Google Gemini Model | AI backend for translation      | (Attached to Basic LLM Chain) | Basic LLM Chain         |                                                                                              |
| Google Sheets      | Google Sheets                   | Append new user ID (registration) | Telegram Trigger         | Telegram               | **USER REGISTRATION:** Auto-registers users who message the bot. Add user Id to Google sheet.|
| Telegram           | Telegram                        | Send welcome message on registration | Google Sheets            | -                     |                                                                                              |
| Google Sheets1     | Google Sheets                   | Read registered users           | Basic LLM Chain          | Send a text message    | **SUBSCRIBER MANAGEMENT:** Reads registered users from Google Sheets                         |
| Send a text message| Telegram                        | Send daily quote message        | Google Sheets1           | -                     | **MESSAGE DELIVERY:** Sends formatted quote to all subscribers                              |
| Telegram Trigger   | Telegram Trigger                | Detect new user messages        | -                       | Google Sheets          |                                                                                              |
| Sticky Note        | Sticky Note                    | Documentation                   | -                       | -                     | ## 📱 DAILY QUOTE BOT WITH AI TRANSLATION ... (long descriptive content)                     |
| Sticky Note1       | Sticky Note                    | Documentation                   | -                       | -                     | **DAILY SCHEDULE:** Triggers once per day to send quotes                                    |
| Sticky Note2       | Sticky Note                    | Documentation                   | -                       | -                     | **QUOTE FETCHING** Gets random quote from ZenQuotes API                                     |
| Sticky Note3       | Sticky Note                    | Documentation                   | -                       | -                     | **AI PROCESSING:** Translates + adds emojis with Gemini                                     |
| Sticky Note4       | Sticky Note                    | Documentation                   | -                       | -                     | **SUBSCRIBER MANAGEMENT:** Reads registered users from Google Sheets                        |
| Sticky Note5       | Sticky Note                    | Documentation                   | -                       | -                     | **MESSAGE DELIVERY:** Sends formatted quote to all subscribers                             |
| Sticky Note6       | Sticky Note                    | Documentation                   | -                       | -                     | **USER REGISTRATION:** Auto-registers users who message the bot. Add user Id to Google sheet.|

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Schedule Trigger node**  
   - Type: Schedule Trigger  
   - Set to trigger once per day (default daily interval)  
   - Connect output to **Set Language** node.

2. **Create the Set Language node**  
   - Type: Set  
   - Add a string field named `Translate_lang` with value `"Persian"`  
   - Connect output to **HTTP Request** node.

3. **Create the HTTP Request node**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://zenquotes.io/api/random`  
   - No authentication or special headers required  
   - Connect output to **Basic LLM Chain** node.

4. **Create the Google Gemini Chat Model node**  
   - Type: Langchain Google Gemini Model  
   - Model Name: `models/gemini-2.5-flash`  
   - Configure credentials with Google PaLM API key  
   - This node acts as an AI backend and will be linked as `ai_languageModel` input to the next node.

5. **Create the Basic LLM Chain node**  
   - Type: Langchain LLM Chain  
   - Use a custom prompt:  
     ```
     First translate the following quote to {{ $('Set Language').item.json.Translate_lang }} then stickerize it using related stickers.

     quote:
     {{ $json.q }}

     Just answer in this response format without any label:
     Stickerized original quote.

     Stickerized translated quote.
     ```  
   - Link `Google Gemini Chat Model` as the AI language model backend  
   - Connect output to **Google Sheets1** node.

6. **Create the Google Sheets1 node**  
   - Type: Google Sheets  
   - Operation: Read (default) entire sheet  
   - Document: Link to your Google Sheet document storing subscribers (Sheet1, gid=0)  
   - Configure OAuth2 credentials with Google Sheets account  
   - Connect output to **Send a text message** node.

7. **Create the Send a text message node**  
   - Type: Telegram  
   - Text: Use MarkdownV2 formatted message:  
     ```
     =*{{ $('HTTP Request').item.json.a.replaceAll(".", "\\.") }}* :

     {{  $('Basic LLM Chain').item.json.text.replaceAll(".", "\\.") }}
     ```  
   - Chat ID: `={{ $json.registered_users }}` (dynamic)  
   - Additional Fields: set `parse_mode` to `MarkdownV2`  
   - Credentials: Telegram Bot Token (your bot token)  
   - No output needed.

8. **Create the Telegram Trigger node**  
   - Type: Telegram Trigger  
   - Listen for `message` updates  
   - Credentials: Telegram Bot Token  
   - Connect output to **Google Sheets** node.

9. **Create the Google Sheets node**  
   - Type: Google Sheets  
   - Operation: Append  
   - Document: Same Google Sheet as above  
   - Sheet: Sheet1 (gid=0)  
   - Map columns:  
     - `registered_users`: `={{ $json.message.from.id }}`  
     - `date`: `={{ $now }}`  
   - Credentials: Google Sheets OAuth2 credentials  
   - Connect output to **Telegram** node.

10. **Create the Telegram node**  
    - Type: Telegram  
    - Text:  
      ```
      👋 Welcome,
      I will send you daily quote.
      ```  
    - Chat ID: `={{ $('Telegram Trigger').item.json.message.from.id }}`  
    - Credentials: Telegram Bot Token  
    - No output needed.

11. **Connect nodes accordingly:**

- Schedule Trigger → Set Language → HTTP Request → Basic LLM Chain → Google Sheets1 → Send a text message  
- Telegram Trigger → Google Sheets → Telegram (Welcome Message)  
- Link Google Gemini Chat Model as AI backend to Basic LLM Chain.

12. **Credential Setup:**

- Telegram API credentials (Telegram Bot Token from @BotFather)  
- Google PaLM API credentials for Google Gemini Chat Model  
- Google Sheets OAuth2 credentials with access to the subscriber spreadsheet

13. **Test the workflow:**

- Send a message to the Telegram bot to register yourself  
- Wait for the daily schedule trigger or trigger manually to send the quote  
- Verify that the quote is fetched, translated, stickerized, and sent correctly in Telegram

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                               | Context or Link                           |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------|
| This workflow automatically sends daily inspirational quotes with AI translation to Telegram subscribers. It uses ZenQuotes API for quotes, Google Gemini AI for translation and stylization, and Google Sheets for subscriber management. | Workflow overview sticky note             |
| Demo bot available on Telegram: [@sgsbot](https://t.me/sgsbot)                                                                                                                                                                            | Telegram bot demo link                     |
| Requires setup of Telegram Bot Token from @BotFather, Google Gemini API key (PaLM API), and Google Sheets API OAuth2 credentials.                                                                                                          | Setup instructions in main sticky note   |
| MarkdownV2 requires escaping special characters such as periods (`.`) to avoid formatting issues in Telegram messages.                                                                                                                    | Important formatting detail               |
| Google Sheets used as a simple database for storing Telegram user IDs and registration dates.                                                                                                                                              | Subscriber data management                |

---

**Disclaimer:**  
The text provided is exclusively derived from an n8n automated workflow. This processing strictly respects applicable content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.