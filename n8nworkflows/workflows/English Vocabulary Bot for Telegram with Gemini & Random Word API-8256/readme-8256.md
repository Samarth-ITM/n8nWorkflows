English Vocabulary Bot for Telegram with Gemini & Random Word API

https://n8nworkflows.xyz/workflows/english-vocabulary-bot-for-telegram-with-gemini---random-word-api-8256


# English Vocabulary Bot for Telegram with Gemini & Random Word API

### 1. Workflow Overview

This workflow automates the process of sending a curated English vocabulary digest message to a Telegram chat every 3 hours. It leverages an external Random Word API to fetch new English words, processes them through Google Gemini AI to generate a vocabulary summary including Vietnamese meanings and example sentences, and finally sends the result as a Telegram message.

The workflow consists of the following logical blocks:  
- **1.1 Scheduled Trigger**: Periodically initiates the workflow every 3 hours.  
- **1.2 Random Word Retrieval and Preparation**: Fetches three random English words and formats them for further processing.  
- **1.3 Vocabulary Summarization via Google Gemini**: Calls the Gemini AI model to create a vocabulary digest message with meanings and example sentences.  
- **1.4 Telegram Message Dispatch**: Sends the AI-generated vocabulary digest to a specified Telegram chat.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger

- **Overview:**  
  This block triggers the entire workflow execution every 3 hours, ensuring periodic updates without manual intervention.

- **Nodes Involved:**  
  - Schedule Trigger

- **Node Details:**  

  - **Schedule Trigger**  
    - *Type & Role:* n8n built-in trigger node for time-based execution.  
    - *Configuration:* Set to trigger every 3 hours (hoursInterval = 3).  
    - *Expressions/Variables:* None.  
    - *Input/Output:* No input; outputs a trigger event to the next node.  
    - *Version-Specific:* Uses typeVersion 1.2, fully compatible with current n8n versions.  
    - *Edge Cases & Failures:* Potential issues if n8n server is down or paused; no retry logic needed as it triggers on schedule.  
    - *Sub-workflow:* None.

#### 1.2 Random Word Retrieval and Preparation

- **Overview:**  
  This block fetches three random English words from an external API and prepares the data to be aggregated and passed to the AI model.

- **Nodes Involved:**  
  - HTTP Request  
  - Edit Fields  
  - Aggregate

- **Node Details:**  

  - **HTTP Request**  
    - *Type & Role:* HTTP request node to call an external REST API.  
    - *Configuration:*  
      - URL: `https://random-word-api.vercel.app/api?words=3`  
      - Method: GET (default)  
      - No authentication or headers specified.  
    - *Expressions/Variables:* None, static URL.  
    - *Input/Output:* Triggered by Schedule Trigger; outputs JSON array of 3 random words.  
    - *Edge Cases & Failures:*  
      - API downtime or rate limiting may cause failure or empty response.  
      - No error handling configured; workflow may fail or pass empty data downstream.  
    - *Sub-workflow:* None.

  - **Edit Fields**  
    - *Type & Role:* Set node to transform incoming data structure for compatibility.  
    - *Configuration:*  
      - Mode: Raw  
      - JSON Output:  
        ```javascript
        {
          "word": "{{ $json }}"
        }
        ```  
      - This wraps each word into an object with a `word` property.  
    - *Expressions/Variables:* Uses `{{ $json }}` to access current item data.  
    - *Input/Output:* Takes array of words as input; outputs array of objects each with a `word` field.  
    - *Edge Cases & Failures:* If input is malformed or empty, output may be incorrect or empty.  
    - *Sub-workflow:* None.

  - **Aggregate**  
    - *Type & Role:* Aggregate node to group all word objects into a single item.  
    - *Configuration:*  
      - Aggregates on the field named `word` to combine multiple items into one array.  
    - *Expressions/Variables:* None.  
    - *Input/Output:* Input: multiple items (each a single word object); output: one item with array of words under `word`.  
    - *Edge Cases & Failures:* No aggregation if input empty; may cause failure downstream if no words are available.  
    - *Sub-workflow:* None.

#### 1.3 Vocabulary Summarization via Google Gemini

- **Overview:**  
  This block sends the aggregated English words to Google Gemini AI (via LangChain node) to generate a vocabulary digest message including Vietnamese meanings and example sentences.

- **Nodes Involved:**  
  - Message a model

- **Node Details:**  

  - **Message a model**  
    - *Type & Role:* LangChain Google Gemini node for AI language model interaction.  
    - *Configuration:*  
      - Model ID: `models/gemini-1.5-flash` (Gemini 1.5 Flash model)  
      - Messages: A single system/user prompt asking the AI to create a vocabulary digest message including English words, Vietnamese meanings, and example sentences. The words are passed dynamically using:  
        ```
        Today's vocabulary list:
        {{ $json.word }}
        ```  
      - No additional options or parameters specified.  
    - *Expressions/Variables:* Uses `{{ $json.word }}` to pass the aggregated words array as part of the prompt.  
    - *Input/Output:* Receives aggregated words; outputs AI-generated message content.  
    - *Credentials:* Requires Google Palm API credentials for authentication.  
    - *Edge Cases & Failures:*  
      - Authentication failure if credentials invalid or expired.  
      - API quota limits or network issues can cause request failure.  
      - AI model may return incomplete or unexpected responses; no validation downstream.  
    - *Sub-workflow:* None.

#### 1.4 Telegram Message Dispatch

- **Overview:**  
  This block sends the AI-generated vocabulary digest as a Telegram text message to a configured chat.

- **Nodes Involved:**  
  - Send a text message

- **Node Details:**  

  - **Send a text message**  
    - *Type & Role:* Telegram node to send messages via Telegram Bot API.  
    - *Configuration:*  
      - Text: Extracts the AI message text from the Gemini node output using expression:  
        ```
        {{ $json.content.parts[0].text }}
        ```  
      - Chat ID: Hardcoded as `"xxxxx"` (to be replaced with actual chat ID).  
      - No additional fields set.  
    - *Expressions/Variables:* Uses dynamic expression to extract AI message text.  
    - *Input/Output:* Input is the AI-generated message; output is the Telegram API response.  
    - *Credentials:* Uses Telegram API credentials named `learn_english_bot`.  
    - *Edge Cases & Failures:*  
      - Invalid chat ID or bot token causes authentication errors.  
      - Network issues or Telegram API downtime can fail message delivery.  
      - Message length or formatting errors may cause API rejection.  
    - *Sub-workflow:* None.

---

### 3. Summary Table

| Node Name          | Node Type                            | Functional Role                         | Input Node(s)           | Output Node(s)      | Sticky Note                                                                                      |
|--------------------|------------------------------------|---------------------------------------|------------------------|---------------------|-------------------------------------------------------------------------------------------------|
| Schedule Trigger    | n8n-nodes-base.scheduleTrigger     | Time-based workflow trigger            | —                      | HTTP Request        |                                                                                                 |
| HTTP Request       | n8n-nodes-base.httpRequest          | Fetch 3 random English words           | Schedule Trigger       | Edit Fields         |                                                                                                 |
| Edit Fields        | n8n-nodes-base.set                  | Wrap words in object with "word" field | HTTP Request           | Aggregate           |                                                                                                 |
| Aggregate          | n8n-nodes-base.aggregate            | Combine individual word items into array | Edit Fields            | Message a model     |                                                                                                 |
| Message a model    | @n8n/n8n-nodes-langchain.googleGemini | Generate vocabulary digest with AI     | Aggregate              | Send a text message |                                                                                                 |
| Send a text message | n8n-nodes-base.telegram             | Send vocabulary digest to Telegram chat | Message a model        | —                   | Replace "xxxxx" in Chat ID with actual Telegram chat ID                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a `Schedule Trigger` node:**  
   - Set its interval to trigger every 3 hours (`hoursInterval = 3`).  
   - No additional configuration needed.

3. **Add an `HTTP Request` node:**  
   - Connect the output of `Schedule Trigger` to this node.  
   - Set method to GET (default).  
   - Set URL to `https://random-word-api.vercel.app/api?words=3`.  
   - No authentication or headers required.

4. **Add a `Set` node named "Edit Fields":**  
   - Connect the output of `HTTP Request` to this node.  
   - Configure mode as Raw.  
   - In JSON Output, enter:  
     ```json
     {
       "word": "{{ $json }}"
     }
     ```  
   - This wraps each word string into an object with a `word` key.

5. **Add an `Aggregate` node:**  
   - Connect the output of "Edit Fields" to this node.  
   - Configure it to aggregate the field named `word` by collecting all words into an array in a single item.  
   - Use default options.

6. **Add a LangChain `Google Gemini` node named "Message a model":**  
   - Connect the output of `Aggregate` to this node.  
   - Configure the model ID to `models/gemini-1.5-flash`.  
   - In the Messages parameter, set a single prompt with the content:  
     ```
     You are a vocabulary-summarizing AI. Please write a vocabulary digest message using the provided English words, including: the English word, the Vietnamese meaning, and one example sentence that contains that English word.
     Today's vocabulary list:
     {{ $json.word }}
     ```  
   - Set appropriate Google Palm API credentials for authentication.

7. **Add a `Telegram` node named "Send a text message":**  
   - Connect the output of "Message a model" to this node.  
   - Set the text parameter using the expression:  
     ```
     {{ $json.content.parts[0].text }}
     ```  
   - Set the Chat ID to your target Telegram chat ID (replace `"xxxxx"` with actual value).  
   - Configure Telegram API credentials for your bot.

8. **Save and activate the workflow.**

---

### 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                            |
|-------------------------------------------------------------------------------------------------|-------------------------------------------|
| Replace `"xxxxx"` in Telegram node's Chat ID with your actual Telegram chat ID or group ID.     | Critical for message delivery correctness |
| Google Gemini API requires valid Google Palm API credentials, ensure quota and permissions.     | https://developers.generativeai.google/   |
| Random Word API used is free and public: `https://random-word-api.vercel.app/`                   | May have rate limits or downtime           |
| The AI prompt instructs Gemini to provide Vietnamese meanings, customize prompt for other languages or details as needed. | Adjust prompt content for localization     |

---

**Disclaimer:** The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All data manipulated is legal and public.