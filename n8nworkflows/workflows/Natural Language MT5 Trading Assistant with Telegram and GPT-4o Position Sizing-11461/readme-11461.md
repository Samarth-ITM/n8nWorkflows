Natural Language MT5 Trading Assistant with Telegram and GPT-4o Position Sizing

https://n8nworkflows.xyz/workflows/natural-language-mt5-trading-assistant-with-telegram-and-gpt-4o-position-sizing-11461


# Natural Language MT5 Trading Assistant with Telegram and GPT-4o Position Sizing

### 1. Workflow Overview

This workflow implements a Natural Language Trading Assistant for MT5 (MetaTrader 5) that interacts with users via Telegram and leverages GPT-4o for advanced position sizing and trade management. It processes user messages, interprets trading instructions using AI language models, queries pending trade signals, places market or limit orders accordingly, and provides detailed feedback to the user.

The workflow is logically divided into the following functional blocks:

- **1.1 Telegram Input Reception:** Captures incoming Telegram chat messages from users.
- **1.2 Preprocessing & AI Classification:** Prepares input data, runs GPT-4o models to classify the user's trading intent, and checks if further AI fallback or data is required.
- **1.3 Trade Signal Retrieval:** Queries backend services to fetch any pending trade signals for the user.
- **1.4 Trade Execution Logic:** Based on AI classification and signal presence, decides the trade type (market or limit), then sends appropriate HTTP requests to MT5 API to place orders.
- **1.5 Response Handling & User Notification:** Parses responses from the trade execution, handles errors or success, and sends Telegram messages back to the user to inform them.
- **1.6 Error and Fallback Handling:** Manages error cases such as missing parameters, HTTP request failures, or AI fallback scenarios with separate Telegram notifications.

---

### 2. Block-by-Block Analysis

#### 2.1 Telegram Input Reception

**Overview:**  
This block captures Telegram chat triggers whenever a user sends a message. It initializes credential and parameter settings for downstream processing.

**Nodes Involved:**  
- Telegram Chat Trigger  
- set credentials and params1

**Node Details:**  

- **Telegram Chat Trigger**  
  - Type: Telegram Trigger  
  - Role: Entry point capturing incoming Telegram messages via webhook.  
  - Config: Uses a specific webhookId linked to the Telegram bot.  
  - Connections: Output → set credentials and params1  
  - Edge Cases: Possible webhook connectivity issues or Telegram API downtime.

- **set credentials and params1**  
  - Type: Set Node  
  - Role: Assigns necessary credentials (e.g., OpenAI API key, MT5 API credentials) and initializes parameters for AI queries and HTTP calls.  
  - Connections: Output → react to message  
  - Edge Cases: Missing or invalid credentials will cause downstream failures.

---

#### 2.2 Preprocessing & AI Classification

**Overview:**  
This block sends the user's message through GPT-4o based chains to classify the intent and generate trading instructions. It checks if the AI output is complete, if fallback LLM calls are needed, and parses the AI responses for further processing.

**Nodes Involved:**  
- react to message (HTTP Request)  
- Trading Assistant Classifier & Responder2 (Langchain LLM)  
- parse LLM output1 (Code)  
- Switch2 (Switch)  
- parse if AI says content is complete2 (Code)  
- needs LLM fallback?1 (Switch)  
- basic LLM fallback check1 (Langchain LLM)  
- parse if AI says content is complete3 (Code)  
- parameters are missing?1 (If)  
- respond: LLM fallback  
- respond: missing params  
- ai agent instructions1 (Set)  
- respond: part 1  
- respond: part 2  
- respond: part 3  
- 4o - mini1, 4o mini1, 4o mini3 (Langchain LLM ChatOpenAI)  
- Trading Assistant Classifier & Responder3 (Langchain LLM)  
- If ai says complete1 (If)  
- If trade was successful1 (If)  
- if there is an error1 (If)  
- parse ai response1 (Code)

**Node Details:**

- **react to message**  
  - Type: HTTP Request  
  - Role: Invokes an external service or API (possibly AI or intermediate service) to initiate classification.  
  - Connections: → Trading Assistant Classifier & Responder2  
  - Edge Cases: HTTP timeouts, malformed responses.

- **Trading Assistant Classifier & Responder2 / 3**  
  - Type: Langchain Chain LLM  
  - Role: Runs GPT-4o chains to classify user instructions and generate trading assistant responses.  
  - Config: Uses OpenAI GPT-4o model with prompt templates tailored to trading.  
  - Connections: Outputs feed into code nodes that parse and decide next steps.  
  - Edge Cases: Rate limits, API key invalidation, unexpected LLM outputs.

- **parse LLM output1 / parse ai response1 / parse if AI says content is complete2 / parse if AI says content is complete3 / Parse switch output1**  
  - Type: Code Node  
  - Role: Custom JavaScript code parses AI JSON/text responses to determine completeness, errors, or actionable signals.  
  - Edge Cases: Parsing errors, malformed JSON from AI, unexpected content structure.

- **Switch2 / Switch3**  
  - Type: Switch  
  - Role: Branch logic based on parsed AI output or HTTP response status to route workflow accordingly.  
  - Edge Cases: Missing or unexpected switch values causing incorrect routing.

- **needs LLM fallback?1 / basic LLM fallback check1**  
  - Type: Switch and Langchain LLM respectively  
  - Role: Determine if the conversation needs fallback to a simpler or different LLM model due to incomplete or failed AI output.  
  - Edge Cases: Infinite fallback loops if fallback criteria are misconfigured.

- **parameters are missing?1**  
  - Type: If  
  - Role: Checks if necessary parameters for trade execution are present; if not, triggers fallback or error responses.  
  - Edge Cases: False negatives causing unnecessary fallback or false positives causing crashes.

- **respond: LLM fallback / respond: missing params / respond: part 1/2/3**  
  - Type: Telegram  
  - Role: Sends segmented or specific fallback or error messages back to the user via Telegram.  
  - Edge Cases: Telegram API outages, message length limits.

- **ai agent instructions1**  
  - Type: Set  
  - Role: Sets instructions or context for the AI agent for better responses.  
  - Edge Cases: Incorrect instructions could degrade AI output quality.

- **4o - mini1 / 4o mini1 / 4o mini3**  
  - Type: Langchain LLM ChatOpenAI  
  - Role: GPT-4o mini variations used for different stages of AI processing or fallback.  
  - Edge Cases: API limits, inconsistent outputs.

---

#### 2.3 Trade Signal Retrieval

**Overview:**  
This block queries an external API endpoint to retrieve any pending trade signals related to the user's account/instructions.

**Nodes Involved:**  
- HTTP get pending signals1  
- if there is signal1  
- respond: No pending signals1  
- respond: No pending signals

**Node Details:**

- **HTTP get pending signals1**  
  - Type: HTTP Request  
  - Role: Queries backend service to fetch pending trade signals.  
  - Config: Uses API URL and authentication credentials set in earlier nodes.  
  - Edge Cases: HTTP errors, no signals returned, response timeouts.

- **if there is signal1**  
  - Type: If  
  - Role: Checks whether returned signal data is non-empty to branch accordingly.  
  - Edge Cases: Incorrect evaluation due to unexpected data formats.

- **respond: No pending signals1 / respond: No pending signals**  
  - Type: Telegram  
  - Role: Notifies user when no pending trade signals exist.  
  - Edge Cases: Telegram delivery failures.

---

#### 2.4 Trade Execution Logic

**Overview:**  
Based on AI classification and retrieved signals, this block decides whether to send a market or limit order, then executes the appropriate HTTP API requests to MT5 or broker API.

**Nodes Involved:**  
- Switch3  
- HTTP Request send market order1  
- HTTP Request send limit order  
- Parse switch output1  
- If trade was successful1  
- respond: trade placement success  
- respond: error in trade placement  
- respond: http req error

**Node Details:**

- **Switch3**  
  - Type: Switch  
  - Role: Determines order type to place based on AI output (market or limit).  
  - Edge Cases: Unknown order types causing failure.

- **HTTP Request send market order1 / HTTP Request send limit order**  
  - Type: HTTP Request  
  - Role: Sends HTTP POST requests to place market or limit orders on MT5 or broker API.  
  - Config: Uses credentials and parameters from earlier nodes.  
  - Edge Cases: API authentication failures, order rejection, network timeouts.

- **Parse switch output1**  
  - Type: Code  
  - Role: Parses API response to extract order confirmation or errors.  
  - Edge Cases: Unexpected response structure.

- **If trade was successful1**  
  - Type: If  
  - Role: Checks if order was accepted and placed successfully.  
  - Edge Cases: False positives/negatives due to API response format changes.

- **respond: trade placement success / respond: error in trade placement / respond: http req error**  
  - Type: Telegram  
  - Role: Sends user feedback about trade execution results or errors.  
  - Edge Cases: Telegram API issues.

---

#### 2.5 Response Handling & User Notification

**Overview:**  
This block handles all Telegram responses to inform the user about status updates, errors, or confirmations.

**Nodes Involved:**  
- respond to query  
- respond: part 1  
- respond: part 2  
- respond: part 3  
- respond: all clear  
- respond: error1  
- respond: No pending signals  
- respond: No pending signals1  
- respond: LLM fallback  
- respond: missing params  
- respond: trade placement success  
- respond: error in trade placement  
- respond: http req error

**Node Details:**

- All Telegram nodes send messages using the Telegram Bot API tied to the same webhookId.  
- Messages are often split into parts (part 1/2/3) to handle longer content.  
- Edge Cases include message delivery failures, rate limiting, or malformed message content.

---

### 3. Summary Table

| Node Name                         | Node Type                 | Functional Role                              | Input Node(s)                      | Output Node(s)                     | Sticky Note                         |
|----------------------------------|---------------------------|----------------------------------------------|----------------------------------|----------------------------------|-----------------------------------|
| Telegram Chat Trigger             | Telegram Trigger          | Entry point for Telegram user messages       | -                                | set credentials and params1       |                                   |
| set credentials and params1       | Set                       | Setup credentials and parameters              | Telegram Chat Trigger             | react to message                  |                                   |
| react to message                 | HTTP Request              | Initial AI classification trigger             | set credentials and params1       | Trading Assistant Classifier & Responder2 |                                   |
| Trading Assistant Classifier & Responder2 | Langchain Chain LLM      | AI classification and response generation     | react to message                  | parse LLM output1                |                                   |
| parse LLM output1               | Code                      | Parse AI output to decide workflow branch     | Trading Assistant Classifier & Responder2 | Switch2                        |                                   |
| Switch2                        | Switch                    | Branch based on AI content completeness       | parse LLM output1                | parse if AI says content is complete2 / respond to query / ai agent instructions1 / HTTP get pending signals1 / Http req clear all signals1 |                                   |
| parse if AI says content is complete2 | Code                      | Parse AI completeness response                 | Switch2                         | needs LLM fallback?1             |                                   |
| needs LLM fallback?1            | Switch                    | Decide if fallback to simpler LLM needed      | parse if AI says content is complete2 | basic LLM fallback check1 / Trading Assistant Classifier & Responder3 / respond: missing params |                                   |
| basic LLM fallback check1       | Langchain Chain LLM       | Run fallback LLM check                          | needs LLM fallback?1             | parse if AI says content is complete3 |                                   |
| parse if AI says content is complete3 | Code                      | Parse fallback LLM completeness check          | basic LLM fallback check1        | parameters are missing?1          |                                   |
| parameters are missing?1         | If                        | Check if required parameters are missing       | parse if AI says content is complete3 | respond: LLM fallback / If ai says complete1 |                                   |
| respond: LLM fallback           | Telegram                  | Notify user about fallback usage                | parameters are missing?1          | -                                |                                   |
| If ai says complete1            | If                        | Branch based on AI completion status           | parameters are missing?1          | if there is an error1 / Trading Assistant Classifier & Responder3 |                                   |
| Trading Assistant Classifier & Responder3 | Langchain Chain LLM      | Secondary AI classification and response       | needs LLM fallback?1 / If ai says complete1 | parse ai response1             |                                   |
| parse ai response1              | Code                      | Parse AI response for errors                     | Trading Assistant Classifier & Responder3 | if there is an error1           |                                   |
| if there is an error1            | If                        | Check for errors in AI or HTTP responses         | parse ai response1              | respond: error1 / Switch3         |                                   |
| Switch3                        | Switch                    | Decide order type and next steps                | if there is an error1            | HTTP Request send market order1 / HTTP Request send limit order / Parse switch output1 / respond: http req error |                                   |
| HTTP Request send market order1 | HTTP Request              | Send market order to MT5/broker API             | Switch3                        | If trade was successful1          |                                   |
| HTTP Request send limit order   | HTTP Request              | Send limit order to MT5/broker API              | Switch3                        | If trade was successful1          |                                   |
| Parse switch output1            | Code                      | Parse response from trade placement API          | Switch3                        | HTTP Request send limit order     |                                   |
| If trade was successful1        | If                        | Check if trade was successfully placed           | HTTP Request send market order1 / HTTP Request send limit order / Parse switch output1 | respond: trade placement success / respond: error in trade placement |                                   |
| respond: trade placement success | Telegram                  | Notify user of successful trade placement         | If trade was successful1         | -                                |                                   |
| respond: error in trade placement | Telegram                  | Notify user of trade placement failure            | If trade was successful1         | -                                |                                   |
| respond: http req error         | Telegram                  | Notify user of HTTP request errors                | Switch3                        | -                                |                                   |
| HTTP get pending signals1       | HTTP Request              | Get pending trade signals for user                | Switch2                        | if there is signal1               |                                   |
| if there is signal1             | If                        | Check if any trade signals exist                   | HTTP get pending signals1        | respond: No pending signals1 / respond: No pending signals |                                   |
| respond: No pending signals1    | Telegram                  | Notify no pending trade signals                     | if there is signal1              | -                                |                                   |
| respond: No pending signals     | Telegram                  | Notify no pending trade signals                     | if there is signal1              | -                                |                                   |
| Http req clear all signals1     | HTTP Request              | Clear all pending trade signals                     | Switch2                        | respond: all clear                |                                   |
| respond: all clear              | Telegram                  | Confirm all signals cleared                          | Http req clear all signals1      | -                                |                                   |
| respond to query               | Telegram                  | General response to user queries                     | Switch2                        | -                                |                                   |
| respond: part 1                | Telegram                  | Part 1 of multi-message response                     | ai agent instructions1           | respond: part 2                  |                                   |
| respond: part 2                | Telegram                  | Part 2 of multi-message response                     | respond: part 1                 | respond: part 3                  |                                   |
| respond: part 3                | Telegram                  | Part 3 of multi-message response                     | respond: part 2                 | -                                |                                   |
| ai agent instructions1         | Set                       | Sets instructions/context for AI agent              | Switch2                        | respond: part 1                  |                                   |

---

### 4. Reproducing the Workflow from Scratch

**Step 1:** Create a **Telegram Chat Trigger** node  
- Configure with your Telegram Bot credentials.  
- Ensure webhook is properly set up to receive messages.

**Step 2:** Add a **Set** node named `set credentials and params1`  
- Configure to assign all necessary credentials (OpenAI API key, MT5 API keys).  
- Set any static parameters needed for AI prompts or HTTP requests.

**Step 3:** Add an **HTTP Request** node `react to message`  
- Configure to call the initial classification or processing API (if external).  
- Connect output from `set credentials and params1` to this node.

**Step 4:** Add **Langchain Chain LLM** node `Trading Assistant Classifier & Responder2`  
- Configure with GPT-4o model and trading-specific prompt templates.  
- Connect output from `react to message`.

**Step 5:** Add a **Code** node `parse LLM output1`  
- Write JS code to parse AI output JSON/text for completeness and intent.  
- Connect output from `Trading Assistant Classifier & Responder2`.

**Step 6:** Add a **Switch** node `Switch2`  
- Branch on parsed completeness and intent flags from AI output.  
- Define multiple outputs for different cases: complete, incomplete, fallback needed, get signals, clear signals.

**Step 7:** For completeness path, connect to **Telegram** node `respond to query`  
- Configure to respond with AI interpretation or answers.

**Step 8:** For fallback path, create **Switch** node `needs LLM fallback?1`  
- Connect output from `parse if AI says content is complete2`.  
- Branch to: fallback LLM call, respond missing params, or continue normal flow.

**Step 9:** Add **Langchain Chain LLM** node `basic LLM fallback check1`  
- Configure with fallback GPT model and prompts.  
- Connect from fallback path in previous step.

**Step 10:** Add **Code** node `parse if AI says content is complete3`  
- Parse fallback output to check for completeness again.

**Step 11:** Add **If** node `parameters are missing?1`  
- Check if required trade parameters exist.  
- Connect to Telegram nodes `respond: LLM fallback` and `respond: missing params` for error handling.

**Step 12:** For complete and valid parameters, connect to **Langchain** node `Trading Assistant Classifier & Responder3`  
- Runs final AI processing before trade execution.

**Step 13:** Add **Code** node `parse ai response1`  
- Parse final AI output for errors or trade instructions.

**Step 14:** Add **If** node `if there is an error1`  
- Branch to Telegram error response or continue trade execution.

**Step 15:** Add **Switch** node `Switch3`  
- Branch based on order type (market or limit) or HTTP error.

**Step 16:** Add two **HTTP Request** nodes:  
- `HTTP Request send market order1` for market orders.  
- `HTTP Request send limit order` for limit orders.  
- Configure with MT5/broker API URL, authentication, and payload from AI outputs.

**Step 17:** Add **Code** node `Parse switch output1`  
- Parse HTTP response for order confirmation.

**Step 18:** Add **If** node `If trade was successful1`  
- Branch to success or error Telegram responses.

**Step 19:** Add Telegram nodes for responses:  
- `respond: trade placement success`  
- `respond: error in trade placement`  
- `respond: http req error`

**Step 20:** Add **HTTP Request** node `HTTP get pending signals1`  
- Configure to query backend for pending trade signals.

**Step 21:** Add **If** node `if there is signal1`  
- Branch to Telegram notifications: `respond: No pending signals1` or `respond: No pending signals`.

**Step 22:** Add **HTTP Request** node `Http req clear all signals1`  
- Configure to clear all pending signals.

**Step 23:** Add Telegram node `respond: all clear`  
- Confirm signal clearance to user.

**Step 24:** Add linked Telegram nodes for multi-part messages (`respond: part 1`, `respond: part 2`, `respond: part 3`)  
- Use to send long messages in sequences.

---

### 5. General Notes & Resources

| Note Content                                                                                                    | Context or Link                                             |
|----------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------|
| The workflow integrates GPT-4o models via Langchain node for enhanced AI-driven trading assistance.            | n8n Langchain Integration Documentation                     |
| Telegram nodes use webhook-based triggers and message sending via Bot Token authentication.                     | Telegram Bot API: https://core.telegram.org/bots/api        |
| MT5 or broker API HTTP requests require valid API keys and endpoint URLs specific to your trading platform.    | MT5 API Docs or Broker API documentation                     |
| Code nodes parse AI JSON responses and must be maintained if prompt output format changes over time.           | Custom JavaScript in n8n Code Nodes                          |
| Multi-part Telegram message sending is used to circumvent message length limits and improve user readability.  | Telegram message size limits and best practices             |

---

*Disclaimer:* The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.