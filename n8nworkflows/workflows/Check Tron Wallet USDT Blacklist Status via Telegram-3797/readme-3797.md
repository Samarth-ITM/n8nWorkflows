Check Tron Wallet USDT Blacklist Status via Telegram

https://n8nworkflows.xyz/workflows/check-tron-wallet-usdt-blacklist-status-via-telegram-3797


# Check Tron Wallet USDT Blacklist Status via Telegram

### 1. Workflow Overview

This workflow enables users to verify whether a Tron blockchain wallet address is blacklisted on the USDT contract by interacting with a Telegram bot. Users send a wallet address via Telegram, and the workflow queries the Tronscan API to check blacklist status, then returns the result to the user.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Validation:** Captures user input from Telegram and verifies the wallet address format.
- **1.2 API Query:** Sends a GET request to the Tronscan API to fetch blacklist data for the provided wallet address.
- **1.3 Response Processing:** Analyzes the API response and formats an appropriate message.
- **1.4 Output Delivery:** Sends the formatted message back to the user through Telegram.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Validation

**Overview:**  
This block listens for incoming messages from Telegram users and validates the format of the wallet address provided to ensure it matches the expected Tron wallet pattern before proceeding.

**Nodes Involved:**  
- Telegram Trigger  
- Check Wallet Address Format (If node)  
- Set Error Message (Wallet Address Format) (Code node)

**Node Details:**

- **Telegram Trigger**  
  - Type: Telegram Trigger  
  - Role: Entry point; listens for new messages from Telegram users.  
  - Configuration:  
    - Listens for "message" update type.  
    - Uses Telegram API credentials for authentication.  
  - Inputs: None (trigger node).  
  - Outputs: Emits incoming Telegram message JSON.  
  - Edge Cases:  
    - Telegram API connectivity issues.  
    - Invalid or missing credentials.  
  - Notes: Captures the entire message object including sender ID and message text.

- **Check Wallet Address Format**  
  - Type: If node  
  - Role: Validates the wallet address format using a regex pattern.  
  - Configuration:  
    - Condition: Checks if the message text matches regex `T[A-Za-z1-9]{33}` (Tron wallet address format).  
    - Case sensitive: true  
    - Type validation: strict string matching.  
  - Inputs: Receives message JSON from Telegram Trigger.  
  - Outputs:  
    - True branch: Valid wallet address format.  
    - False branch: Invalid format.  
  - Edge Cases:  
    - User sends empty or malformed input.  
    - Regex mismatch leading to false negatives.  
  - Version: 2.2 (supports advanced conditions).

- **Set Error Message (Wallet Address Format)**  
  - Type: Code node  
  - Role: Generates an error message if wallet address format is invalid.  
  - Configuration:  
    - JavaScript code returns a JSON object with a user-friendly error text: "Please enter your wallet address correctly and completely."  
  - Inputs: Triggered from the false branch of the If node.  
  - Outputs: JSON with error message text.  
  - Edge Cases: Minimal; code is simple and deterministic.

---

#### 1.2 API Query

**Overview:**  
This block sends a GET request to the Tronscan API to retrieve blacklist information for the validated wallet address.

**Nodes Involved:**  
- Tron BlackList Stable Token Api Request (HTTP Request node)

**Node Details:**

- **Tron BlackList Stable Token Api Request**  
  - Type: HTTP Request  
  - Role: Queries the Tronscan API for blacklist status of the wallet address.  
  - Configuration:  
    - Method: GET  
    - URL: `https://apilist.tronscanapi.com/api/stableCoin/blackList?blackAddress={{ $json.message.text }}`  
    - Response format: JSON (default)  
  - Inputs: Receives wallet address from the true branch of the If node.  
  - Outputs: API response JSON containing blacklist data.  
  - Edge Cases:  
    - API downtime or network errors.  
    - Rate limiting by Tronscan API.  
    - Unexpected or malformed API responses.  
  - Notes: Uses dynamic expression to insert user-provided wallet address into URL.

---

#### 1.3 Response Processing

**Overview:**  
Processes the API response to determine if the wallet is blacklisted and formats a user-friendly message accordingly.

**Nodes Involved:**  
- Check Api Response (Code node)

**Node Details:**

- **Check Api Response**  
  - Type: Code node  
  - Role: Parses API response and constructs the message to send back.  
  - Configuration:  
    - JavaScript code inspects `response.total` field:  
      - If `total > 0`, constructs a warning message indicating the wallet is blacklisted, including the blacklisted address.  
      - Else, constructs a confirmation message indicating the wallet is not blacklisted.  
  - Inputs: Receives API response JSON from HTTP Request node.  
  - Outputs: JSON containing the formatted text message for Telegram.  
  - Edge Cases:  
    - Missing or unexpected API response structure.  
    - Empty or zero `total` field.  
    - Potential JavaScript runtime errors if response is undefined or malformed.

---

#### 1.4 Output Delivery

**Overview:**  
Sends the formatted message back to the user via the Telegram bot, replying to the original message.

**Nodes Involved:**  
- Telegram Send Message

**Node Details:**

- **Telegram Send Message**  
  - Type: Telegram node  
  - Role: Sends the response message to the Telegram user.  
  - Configuration:  
    - Resource: Message  
    - Operation: Send  
    - Chat ID: Extracted dynamically from the original Telegram message sender ID (`{{$json["chat_id"]}}` or `={{ $('Telegram Trigger').item.json.message.from.id }}`)  
    - Text: Uses the formatted message text from the previous node (`={{ $json.text }}`)  
    - Additional Fields: Replies to the original message ID to maintain context.  
  - Inputs: Receives JSON with text and chat ID.  
  - Outputs: None (terminal node).  
  - Edge Cases:  
    - Telegram API errors (invalid token, chat ID issues).  
    - Message length limits or formatting issues.  
  - Notes: Uses same Telegram API credentials as the trigger node.

---

### 3. Summary Table

| Node Name                        | Node Type               | Functional Role                          | Input Node(s)               | Output Node(s)                     | Sticky Note                                                                                  |
|---------------------------------|-------------------------|----------------------------------------|-----------------------------|----------------------------------|----------------------------------------------------------------------------------------------|
| Telegram Trigger                | Telegram Trigger        | Receives Telegram messages              | None                        | Check Wallet Address Format       |                                                                                              |
| Check Wallet Address Format     | If                      | Validates Tron wallet address format    | Telegram Trigger            | Tron BlackList Stable Token Api Request (true), Set Error Message (false) |                                                                                              |
| Set Error Message (Wallet Address Format) | Code                    | Generates error message for invalid input | Check Wallet Address Format (false) | Telegram Send Message             |                                                                                              |
| Tron BlackList Stable Token Api Request | HTTP Request            | Queries Tronscan API for blacklist info | Check Wallet Address Format (true) | Check Api Response               |                                                                                              |
| Check Api Response              | Code                    | Processes API response and formats message | Tron BlackList Stable Token Api Request | Telegram Send Message             |                                                                                              |
| Telegram Send Message           | Telegram                | Sends message back to Telegram user     | Set Error Message (Wallet Address Format), Check Api Response | None                             |                                                                                              |
| Sticky Note                    | Sticky Note             | Provides workflow description            | None                        | None                             | ## TRON USDT Blacklist Checker<br>**This template checks USDT wallets on the TRON blockchain and queries whether they have been blacklisted.** |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node**  
   - Type: Telegram Trigger  
   - Configure to listen for "message" update type.  
   - Set Telegram API credentials (create a Telegram bot and add credentials in n8n).  
   - Position: Start of workflow.

2. **Add If Node: Check Wallet Address Format**  
   - Type: If  
   - Condition: String matches regex `T[A-Za-z1-9]{33}` on `{{$json.message.text}}`  
   - Case sensitive: true  
   - Connect Telegram Trigger output to this node input.

3. **Add Code Node: Set Error Message (Wallet Address Format)**  
   - Type: Code  
   - JavaScript code:  
     ```js
     return [
       {
         json: {
           text: 'Please enter your wallet address correctly and completely.',
         },
       },
     ];
     ```  
   - Connect the false output of the If node (invalid format) to this node.

4. **Add HTTP Request Node: Tron BlackList Stable Token Api Request**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://apilist.tronscanapi.com/api/stableCoin/blackList?blackAddress={{ $json.message.text }}`  
   - Connect the true output of the If node (valid format) to this node.

5. **Add Code Node: Check Api Response**  
   - Type: Code  
   - JavaScript code:  
     ```js
     const response = items[0].json;
     let message;

     if (response.total && response.total > 0) {
       message = `🚨🛑 **This Wallet is Blacklisted!** 🛑🚨: ${response.data[0].blackAddress}`;
     } else {
       message = `✅💚 **This Wallet is NOT Blacklisted!** 💚✅.`;
     }

     return [
       {
         json: {
           text: message,
         },
       },
     ];
     ```  
   - Connect HTTP Request node output to this node.

6. **Add Telegram Send Message Node**  
   - Type: Telegram  
   - Resource: Message  
   - Operation: Send  
   - Chat ID: `={{ $('Telegram Trigger').item.json.message.from.id }}`  
   - Text: `={{ $json.text }}`  
   - Additional Fields: Set `reply_to_message_id` to `={{ $('Telegram Trigger').item.json.message.message_id }}` to reply to original message.  
   - Set Telegram API credentials (same as Telegram Trigger).  
   - Connect outputs of both the "Set Error Message" node and "Check Api Response" node to this node.

7. **Workflow Activation**  
   - Save and activate the workflow.  
   - Ensure Telegram bot token is valid and webhook is properly set up.

---

### 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                                                                                  |
|-------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| This workflow provides a simple and efficient way to check the blacklist status of Tron wallet addresses via Telegram. | Workflow description and use case.                                                             |
| Telegram bot setup requires creating a bot via BotFather and obtaining API token.                | Telegram official documentation: https://core.telegram.org/bots/api                              |
| Tron wallet address format regex: `T[A-Za-z1-9]{33}`                                            | Ensures only valid Tron addresses are processed.                                               |
| Tronscan API endpoint used: https://apilist.tronscanapi.com/api/stableCoin/blackList            | Official Tronscan API for stablecoin blacklist checking.                                       |
| The workflow replies to the user's original Telegram message to maintain conversation context.  | Improves user experience in Telegram chats.                                                    |

---

This document fully describes the "Telegram Tron Wallet Blacklist Checker" workflow, enabling advanced users and AI agents to understand, reproduce, and modify the workflow confidently.