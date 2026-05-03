Send Daily Mortgage Rate Updates from Mortgage News Daily to Messaging Platforms

https://n8nworkflows.xyz/workflows/send-daily-mortgage-rate-updates-from-mortgage-news-daily-to-messaging-platforms-8988


# Send Daily Mortgage Rate Updates from Mortgage News Daily to Messaging Platforms

### 1. Workflow Overview

This workflow automates the process of sending daily mortgage rate updates sourced from Mortgage News Daily to Discord, with AI-generated client-friendly messages formatted as text and email. It is designed for real estate agents, lenders, or mortgage brokers who want to keep their clients informed with timely rate changes and personalized interpretations.

The workflow consists of four main logical blocks:

- **1.1 Schedule Trigger:** Defines when the workflow runs during the day to fetch updates multiple times daily.
- **1.2 Data Acquisition:** Fetches the current mortgage rates page HTML from Mortgage News Daily.
- **1.3 Data Processing & AI Messaging:** Parses the HTML to extract mortgage rate data, formats it for messaging, then sends the raw data to an AI model to generate client-appropriate text and email messages.
- **1.4 Distribution:** Sends the final AI-generated message to a Discord channel via webhook for client communication.

---

### 2. Block-by-Block Analysis

#### 2.1 Schedule Trigger

- **Overview:**  
  Initiates the workflow execution at defined times throughout the day to provide multiple mortgage rate updates.

- **Nodes Involved:**  
  - Schedule Trigger

- **Node Details:**  
  - **Schedule Trigger**  
    - **Type & Role:** Cron-based trigger node that starts the workflow automatically.  
    - **Configuration:** Set to trigger at 07:30, 10:30, 12:30, 16:30, and 18:30 daily using cron expression `30 7,10,12,16,18 * * *`.  
    - **Inputs:** None (trigger node).  
    - **Outputs:** Triggers the HTTP Request node.  
    - **Edge Cases:** If the n8n instance is offline or the node fails, updates will be missed. Cron misconfiguration may cause no or excessive triggers.  
    - **Sticky Note:** "## 1. Daily Rate Updates\n**Set your Trigger Times Here** Currently you will get a few updates every day!"

#### 2.2 Data Acquisition

- **Overview:**  
  Retrieves the Mortgage News Daily homepage HTML to extract mortgage rate tables.

- **Nodes Involved:**  
  - HTTP Request

- **Node Details:**  
  - **HTTP Request**  
    - **Type & Role:** HTTP GET request node fetching the mortgage rates webpage.  
    - **Configuration:**  
      - URL: `https://www.mortgagenewsdaily.com/`  
      - Custom headers mimicking a browser user agent and typical Accept headers to avoid blocking or receiving incomplete content.  
      - Response expected as raw HTML (not JSON).  
    - **Input:** Triggered by Schedule Trigger.  
    - **Output:** Passes raw HTML content to "Code in JavaScript" node.  
    - **Edge Cases:**  
      - Network errors or site downtime causing fetch failures.  
      - Changes in website structure or anti-scraping mechanisms may block or alter response.  
      - If headers are insufficient, server may return errors or incomplete data.  
    - **Sticky Note:** "## 2. Get Mortgage Rates From Mortgage News Daily \n\nCan change Source if needed but haven't had any issues"

#### 2.3 Data Processing & AI Messaging

- **Overview:**  
  Parses the mortgage rates HTML to extract structured data, formats it into a message for Discord, then sends this raw data to an AI model to generate formatted client messages (text and email).

- **Nodes Involved:**  
  - Code in JavaScript  
  - Message a model

- **Node Details:**  
  - **Code in JavaScript**  
    - **Type & Role:** Custom code node that parses HTML and creates a formatted summary message string.  
    - **Configuration:**  
      - Extracts the "Last Updated" date from the HTML.  
      - Uses regex to extract table rows and cells for mortgage product, rate, change, and points.  
      - Constructs a multi-line string with mortgage rate details formatted for Discord.  
      - Outputs a JSON object with key `content` containing the message.  
    - **Input:** Receives raw HTML from HTTP Request.  
    - **Output:** Passes formatted message JSON to "Message a model".  
    - **Edge Cases:**  
      - HTML structure changes could break regex parsing and cause empty or malformed data.  
      - Missing or malformed "Last Updated" date will default to "Unknown".  
      - If no valid rows found, message will be incomplete.  
    - **Sticky Note:** "## 3. Clean up data for discord message\n\nMight need some updates if you wanted it to go to slack or something have not tested."  

  - **Message a model**  
    - **Type & Role:** AI language model node (Google Gemini) generating client-facing messages.  
    - **Configuration:**  
      - Uses Google Gemini 2.0 Flash Lite model.  
      - Input prompt includes mortgage rate message from previous node and instructs to create two outputs: one formatted for text messaging, one for email, without client names or personal references.  
      - No additional options or customizations.  
    - **Input:** Receives JSON with formatted mortgage rates.  
    - **Output:** Sends AI-generated content to Discord node.  
    - **Credentials:** Google Palm API key required.  
    - **Edge Cases:**  
      - API quota exceeded or invalid API key causes failure.  
      - AI model understanding issues if prompt or input data malformed.  
      - Latency or timeouts from AI service.  
    - **Sticky Note:** "## 3. AI Magic to create Text/Emails to send to clients (Fee API Key at https://aistudio.google.com/api-keys)\n\nThis can be customized if you need different messaging style or if you want to include a name variable for a CRM etc."

#### 2.4 Distribution

- **Overview:**  
  Sends the AI-generated message content to a Discord channel using a webhook for delivery to clients or team members.

- **Nodes Involved:**  
  - Discord

- **Node Details:**  
  - **Discord**  
    - **Type & Role:** Discord node posts messages via webhook authentication.  
    - **Configuration:**  
      - Sends content from "Message a model" node's output JSON `content`.  
      - Includes a placeholder for "Custom Client Messages," which is linked to the AI-generated content parts (though the expression contains `$json.content.parts[0].text` which depends on AI output structure).  
      - Uses stored Discord webhook credentials.  
    - **Input:** Receives AI-generated messages from the "Message a model" node.  
    - **Output:** Final output node (no further connections).  
    - **Credentials:** Discord webhook API credentials required.  
    - **Edge Cases:**  
      - Invalid or revoked webhook URL causes message send failure.  
      - Discord API rate limits or downtime.  
      - Incorrect message formatting or missing fields may cause errors or silent failures.  
    - **Sticky Note:** "## 4. Send cleaned mortgage rates and custom messages for lenders or real estate agents to send to their clients, delivered to Discord Webhook\n\nCould be changed to slack/telegram/whatsapp etc."

---

### 3. Summary Table

| Node Name          | Node Type                    | Functional Role                       | Input Node(s)     | Output Node(s)    | Sticky Note                                                                                           |
|--------------------|------------------------------|-------------------------------------|-------------------|-------------------|-----------------------------------------------------------------------------------------------------|
| Schedule Trigger    | scheduleTrigger              | Initiates workflow on schedule      | -                 | HTTP Request      | ## 1. Daily Rate Updates **Set your Trigger Times Here** Currently you will get a few updates every day! |
| HTTP Request       | httpRequest                  | Fetch mortgage rates HTML page      | Schedule Trigger  | Code in JavaScript | ## 2. Get Mortgage Rates From Mortgage News Daily Can change Source if needed but haven't had any issues |
| Code in JavaScript  | code                        | Parse HTML & format mortgage data   | HTTP Request      | Message a model   | ## 3. Clean up data for discord message Might need some updates if you wanted it to go to slack or something have not tested. |
| Message a model     | @n8n/n8n-nodes-langchain.googleGemini | Generate client-friendly messages  | Code in JavaScript | Discord           | ## 3. AI Magic to create Text/Emails to send to clients (Fee API Key at https://aistudio.google.com/api-keys) This can be customized if you need different messaging style or if you want to include a name variable for a CRM etc. |
| Discord             | discord                     | Send final message to Discord       | Message a model   | -                 | ## 4. Send cleaned mortgage rates and custom messages for lenders or real estate agents to send to their clients, delivered to Discord Webhook Could be changed to slack/telegram/whatsapp etc. |

---

### 4. Reproducing the Workflow from Scratch

1. **Add a Schedule Trigger node:**  
   - Set the trigger to run at 07:30, 10:30, 12:30, 16:30, and 18:30 daily using the cron expression: `30 7,10,12,16,18 * * *`.

2. **Add an HTTP Request node:**  
   - Connect it to the Schedule Trigger node.  
   - Set HTTP Method to GET.  
   - Set URL to `https://www.mortgagenewsdaily.com/`.  
   - Under headers, specify JSON headers with:  
     ```json
     {
       "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/124.0.0.0 Safari/537.36",
       "Accept": "text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8",
       "Accept-Language": "en-US,en;q=0.9",
       "Connection": "keep-alive"
     }
     ```  
   - Enable "Send Headers".

3. **Add a Code node (JavaScript):**  
   - Connect it to the HTTP Request node.  
   - Paste the JavaScript code to parse the HTML:  
     - Extract the "Last Updated" date string.  
     - Use regex to extract table rows and parse mortgage product, rate, change, and points.  
     - Construct a formatted multi-line string for Discord.  
     - Return JSON with `content` key containing the message string.

4. **Add a Google Gemini (PaLM) AI model node:**  
   - Connect it to the Code node.  
   - Choose the model ID: `models/gemini-2.0-flash-lite`.  
   - Configure the prompt/message input with the instruction:  
     > "Please look at the following mortgage rate updates and generate the viewer a message that can be sent to clients and what this could mean for them. Never include a name or things like client, just say something like hello or hi if needed. Give one formatted for Text, one formatted for email. Only output the messages. Rates: {{ $json.content }}"  
   - Set up credentials with a valid Google Palm API key.

5. **Add a Discord node:**  
   - Connect it to the AI model node.  
   - Set "Authentication" to webhook.  
   - Use stored Discord webhook credentials.  
   - Set message content to:  
     ```
     ={{ $('Message a model').item.json.content }}

     Custom Client Messages:
     {{ $json.content.parts[0].text }}
     ```  
     (Adjust expressions if the AI output structure differs.)  

6. **Verify all connections:**  
   - Schedule Trigger → HTTP Request → Code in JavaScript → Message a model → Discord

7. **Set up credentials:**  
   - Google Palm API credentials in the AI node.  
   - Discord webhook credentials in the Discord node.

8. **Test the workflow manually:**  
   - Run the trigger manually to verify data fetching, parsing, AI message generation, and Discord posting.

---

### 5. General Notes & Resources

| Note Content                                                                                               | Context or Link                                         |
|------------------------------------------------------------------------------------------------------------|--------------------------------------------------------|
| Fee API Key for Google Gemini AI is available at https://aistudio.google.com/api-keys                      | AI model credentials setup                              |
| Workflow designed for Discord but can be adapted for Slack, Telegram, WhatsApp by modifying the final node | Distribution flexibility                                |
| JavaScript parsing depends heavily on current Mortgage News Daily HTML structure; monitor for website changes | Maintenance note                                        |
| Cron expression can be customized to adjust update frequency                                              | Scheduling customization                                |

---

**Disclaimer:** The provided text comes exclusively from a workflow automated with n8n, an integration and automation tool. This process strictly adheres to current content policies and contains no illegal, offensive, or protected content. All processed data is legal and public.