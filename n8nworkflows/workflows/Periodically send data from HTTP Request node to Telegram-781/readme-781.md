Periodically send data from HTTP Request node to Telegram

https://n8nworkflows.xyz/workflows/periodically-send-data-from-http-request-node-to-telegram-781


# Periodically send data from HTTP Request node to Telegram

---

### 1. Workflow Overview

This workflow automates the daily sending of a random cocktail recipe to a Telegram chat. It periodically fetches cocktail data from a public API and posts the drink’s image and preparation instructions as a photo message on Telegram.

Logical blocks:

- **1.1 Scheduled Trigger:** Initiates the workflow daily at a specified hour.
- **1.2 Data Retrieval:** Requests a random cocktail recipe from a public API.
- **1.3 Telegram Messaging:** Sends the cocktail image and instructions to a Telegram chat.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger

- **Overview:**  
  This block triggers the workflow once every day at a specified time (20:00 hour).

- **Nodes Involved:**  
  - Cron

- **Node Details:**

  **Cron**  
  - Type and Technical Role: Cron Trigger; initiates the workflow based on time schedule.  
  - Configuration: Configured to trigger daily at 20:00 (8 PM) local time.  
  - Key Expressions/Variables: None; static time trigger.  
  - Input/Output Connections: No input; output connects to HTTP Request node.  
  - Version-specific Requirements: Requires n8n version supporting Cron node v1.  
  - Potential Failures: Misconfiguration of time zone or schedule; node failure unlikely.  
  - Sub-workflow: None.

#### 1.2 Data Retrieval

- **Overview:**  
  Fetches a random cocktail recipe from TheCocktailDB API.

- **Nodes Involved:**  
  - HTTP Request

- **Node Details:**

  **HTTP Request**  
  - Type and Technical Role: HTTP Request node; performs REST API GET request.  
  - Configuration:  
    - URL: https://www.thecocktaildb.com/api/json/v1/1/random.php  
    - HTTP Method: GET (default)  
    - Options: Default (no special headers or parameters)  
  - Key Expressions/Variables: None directly inside node; output JSON expected to contain a `drinks` array with recipe data.  
  - Input/Output Connections: Input from Cron node; output to Telegram node.  
  - Version-specific Requirements: Requires n8n version supporting HTTP Request node v1.  
  - Potential Failures:  
    - Network issues causing request failure or timeout.  
    - API downtime or rate limits.  
    - Unexpected data structure changes in API response.  
  - Sub-workflow: None.

#### 1.3 Telegram Messaging

- **Overview:**  
  Sends the fetched cocktail recipe’s image and instructions as a photo message to a Telegram chat.

- **Nodes Involved:**  
  - Telegram

- **Node Details:**

  **Telegram**  
  - Type and Technical Role: Telegram node; sends messages via Telegram Bot API.  
  - Configuration:  
    - Operation: sendPhoto  
    - Chat ID: -485396236 (target Telegram chat or group)  
    - File: Uses expression to extract the first drink’s image URL from HTTP Request output: `{{$node["HTTP Request"].json["drinks"][0]["strDrinkThumb"]}}`  
    - Caption: Uses expression to extract the preparation instructions: `{{$node["HTTP Request"].json["drinks"][0]["strInstructions"]}}`  
    - Credentials: Uses “telegramApi” credential named “telegram-bot” for authentication.  
  - Key Expressions/Variables: Extracts image URL and instructions dynamically from previous node output.  
  - Input/Output Connections: Input from HTTP Request node; no further output (terminal node).  
  - Version-specific Requirements: Requires Telegram node v1; valid Telegram Bot credentials setup.  
  - Potential Failures:  
    - Invalid or expired Telegram bot token leading to authentication errors.  
    - Incorrect chat ID causing message delivery failure.  
    - Invalid or missing image URL or caption causing message send errors.  
    - Telegram API rate limits or outages.  
  - Sub-workflow: None.

---

### 3. Summary Table

| Node Name     | Node Type           | Functional Role               | Input Node(s) | Output Node(s) | Sticky Note                                                      |
|---------------|---------------------|------------------------------|---------------|----------------|-----------------------------------------------------------------|
| Cron          | Cron Trigger        | Scheduled daily trigger       | -             | HTTP Request   |                                                                 |
| HTTP Request  | HTTP Request        | Fetches random cocktail data | Cron          | Telegram       |                                                                 |
| Telegram      | Telegram            | Sends cocktail photo & text  | HTTP Request  | -              |                                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Cron Node:**  
   - Add a new node of type **Cron**.  
   - Configure it to trigger **daily** at hour **20** (8 PM). No minutes specified means default to 0.  
   - This node has no credentials.  

2. **Create HTTP Request Node:**  
   - Add a new node of type **HTTP Request**.  
   - Set the URL to: `https://www.thecocktaildb.com/api/json/v1/1/random.php`  
   - Method: GET (default).  
   - Leave other options default (no authentication or headers).  
   - Connect Cron node’s output to this node’s input.  

3. **Create Telegram Node:**  
   - Add a new node of type **Telegram**.  
   - Under **Credentials**, create or select your Telegram Bot API credential (OAuth or token-based).  
   - Set **Operation** to `sendPhoto`.  
   - Set **Chat ID** to the target chat/group ID (e.g., `-485396236`).  
   - For **File**, enter the expression: `{{$node["HTTP Request"].json["drinks"][0]["strDrinkThumb"]}}` to dynamically get the drink image URL.  
   - For **Caption**, enter the expression: `{{$node["HTTP Request"].json["drinks"][0]["strInstructions"]}}` to include preparation instructions.  
   - Connect HTTP Request node’s output to Telegram node’s input.  

4. **Activate Workflow:**  
   - Save and activate the workflow.  
   - Confirm Telegram bot has permission to send messages in the target chat.  

---

### 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                                      |
|-------------------------------------------------------------------------------------------------|-----------------------------------------------------|
| TheCocktailDB API provides free JSON endpoints for cocktail data, including random selection.   | https://www.thecocktaildb.com/api.php               |
| Telegram Bot API requires bot token and chat ID; negative chat IDs indicate groups/supergroups. | https://core.telegram.org/bots/api                  |
| Ensure the Telegram bot is added to the chat and has permission to send messages/photos.         | Telegram group settings                              |

---

This document fully describes the workflow’s structure, logic, and configuration, enabling reimplementation or modification with full understanding of potential failure points and integration requirements.