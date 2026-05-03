Delete Dialogflow Intents via Telegram Commands

https://n8nworkflows.xyz/workflows/delete-dialogflow-intents-via-telegram-commands-7409


# Delete Dialogflow Intents via Telegram Commands

### 1. Workflow Overview

This workflow enables deletion of Dialogflow intents through Telegram commands. It is designed to allow a specific Telegram user to send a command (keyword "delete") that triggers the deletion of a Dialogflow intent identified by its unique ID embedded in the HTTP request URLs. The workflow includes validation of the user and command keyword, retrieval of the intent details for confirmation, execution of the delete operation via the Dialogflow API, and confirmation feedback sent back to the Telegram user.

Logical blocks:

- **1.1 Input Reception and Validation:** Receives Telegram messages, validates the user ID and the command keyword.
- **1.2 Intent Retrieval:** Fetches full intent details from Dialogflow using the intent ID for confirmation.
- **1.3 Intent Deletion:** Deletes the specified intent via Dialogflow API.
- **1.4 Confirmation Messaging:** Sends feedback messages to the Telegram user about the success or failure of the operation.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Validation

**Overview:**  
This block handles the reception of Telegram messages, verifies that the sender is an authorized user, and checks that the command keyword is valid.

**Nodes Involved:**  
- Telegram Trigger  
- User validation by ID (IF node)  
- Keyword validation (IF node)  
- Invalid user message (Telegram node)  
- Invalid keyword message (Telegram node)

**Node Details:**

- **Telegram Trigger**  
  - *Type:* Trigger node for Telegram messages  
  - *Configuration:* Listens for "message" updates only  
  - *Expressions:* None  
  - *Input/Output:* No input; outputs Telegram message JSON to next node  
  - *Edge cases:* Missing or malformed messages; connection issues with Telegram API  
  - *Notes:* Entry point of the workflow

- **User validation by ID (IF node)**  
  - *Type:* Conditional node  
  - *Configuration:* Checks if the Telegram message sender’s ID equals a predefined user ID (`Telegram_user_ID`)  
  - *Expressions:* `{{$json.message.from.id}} == Telegram_user_ID`  
  - *Input:* Telegram Trigger  
  - *Output:* Passes valid user messages to Keyword validation; invalid users trigger Invalid user message node  
  - *Edge cases:* User ID missing or mismatched; expression evaluation failures  
  - *Notes:* Ensures only authorized users can delete intents

- **Keyword validation (IF node)**  
  - *Type:* Conditional node  
  - *Configuration:* Checks if the message text equals "delete" (case sensitive)  
  - *Expressions:* `{{$json.message.text}} == "delete"`  
  - *Input:* User validation by ID (valid branch)  
  - *Output:* Passes valid keyword to Intent Retrieval block; invalid keyword triggers Invalid keyword message  
  - *Edge cases:* Case sensitivity; empty or missing message text

- **Invalid user message (Telegram node)**  
  - *Type:* Telegram message node  
  - *Configuration:* Sends message "Invalid user\nUsuario inválido" to the Telegram user ID extracted from the incoming message  
  - *Expressions:* Uses `{{$json.message.from.id}}` for chatId  
  - *Input:* User validation by ID (invalid branch)  
  - *Edge cases:* Telegram API errors; chatId missing

- **Invalid keyword message (Telegram node)**  
  - *Type:* Telegram message node  
  - *Configuration:* Sends "Invalid word\nPalabra inválida" to the Telegram user ID  
  - *Expressions:* Uses `{{$json.message.from.id}}` for chatId  
  - *Input:* Keyword validation (invalid branch)  
  - *Edge cases:* Telegram API errors

---

#### 1.2 Intent Retrieval

**Overview:**  
Retrieves full Dialogflow intent details using a fixed intent ID and project ID to confirm intent existence and obtain its name for user feedback.

**Nodes Involved:**  
- HTTP Request GET NAME  
- Sticky Note (Get the intent name...)  
- Sticky Note (Get the full intent...)  
- Sticky Note5 (Instructions on intent ID URL)  
- Sticky Note4 (Intent name for confirmation message)  
- simulated delay (Wait node)  
- Sticky Note6 (Delay explanation)

**Node Details:**

- **HTTP Request GET NAME**  
  - *Type:* HTTP Request node  
  - *Configuration:* GET request to Dialogflow API URL for specific intent ID (hardcoded URL) with query parameter `intentView=INTENT_VIEW_FULL`  
  - *Authentication:* Uses Google API credentials (OAuth2)  
  - *Expressions:* None dynamic in URL, fixed intent and project ID in URL  
  - *Input:* Keyword validation (valid branch)  
  - *Output:* JSON of the intent details (including displayName)  
  - *Edge cases:* Authentication failure, network timeouts, invalid intent ID, API errors  
  - *Notes:* Critical to obtain intent name before deletion

- **simulated delay (Wait node)**  
  - *Type:* Wait  
  - *Configuration:* Waits 1 second before proceeding to deletion  
  - *Input:* HTTP Request GET NAME output  
  - *Output:* Delays flow to ensure sequential processing  
  - *Edge cases:* Minimal risk, but possible workflow timeouts if many executions

- **Sticky Notes**  
  - Provide instructional context about:
    - How to extract the Intent ID from Dialogflow URLs  
    - The purpose of retrieving the full intent JSON  
    - Reason for the 1-second delay before deletion  
    - Use of the intent name in confirmation messages

---

#### 1.3 Intent Deletion

**Overview:**  
Deletes the targeted intent from Dialogflow using the API DELETE method after successful retrieval and delay.

**Nodes Involved:**  
- HTTP Request ELIMINAR

**Node Details:**

- **HTTP Request ELIMINAR**  
  - *Type:* HTTP Request node  
  - *Configuration:* DELETE request to Dialogflow API URL with hardcoded project and intent IDs  
  - *Authentication:* Uses Google API credentials (OAuth2)  
  - *Input:* simulated delay (post intent retrieval)  
  - *Output:* No data output (alwaysOutputData=false)  
  - *Edge cases:* API authorization errors, invalid intent ID, network issues, rate limiting  
  - *Notes:* Performs the actual deletion of the intent

---

#### 1.4 Confirmation Messaging

**Overview:**  
Sends a confirmation message to the Telegram user confirming the intent has been deleted, including the intent’s display name.

**Nodes Involved:**  
- Confirmation message (Telegram node)  
- Sticky Note1 (Confirmation message explanation)  
- Sticky Note2 (Intent removal confirmation)

**Node Details:**

- **Confirmation message (Telegram node)**  
  - *Type:* Telegram message node  
  - *Configuration:* Sends a message confirming deletion: "The intent: *<intent_name>* has been removed."  
  - *Expressions:* Uses `{{$('HTTP Request GET NAME').item.json.displayName}}` to insert intent name dynamically  
  - *Chat ID:* Extracted from Telegram Trigger node’s message sender ID  
  - *Input:* HTTP Request ELIMINAR (successful branch)  
  - *Edge cases:* Telegram API failures, missing displayName if HTTP Request GET NAME failed silently  
  - *Notes:* `executeOnce` set to true to avoid multiple messages if multiple items passed

---

### 3. Summary Table

| Node Name             | Node Type               | Functional Role                | Input Node(s)          | Output Node(s)          | Sticky Note                                                                                                            |
|-----------------------|-------------------------|-------------------------------|-----------------------|------------------------|------------------------------------------------------------------------------------------------------------------------|
| Telegram Trigger      | telegramTrigger          | Entry point, receive Telegram messages | None                  | User validation by ID   |                                                                                                                        |
| User validation by ID | IF node                 | Validates authorized Telegram user      | Telegram Trigger      | Keyword validation, Invalid user message | The user and the submitted keyword are validated.                                                                      |
| Keyword validation    | IF node                 | Validates the command keyword ("delete") | User validation by ID | HTTP Request GET NAME, Invalid keyword message |                                                                                                                        |
| Invalid user message  | Telegram message        | Sends invalid user warning     | User validation by ID  | None                   |                                                                                                                        |
| Invalid keyword message | Telegram message      | Sends invalid keyword warning  | Keyword validation     | None                   |                                                                                                                        |
| HTTP Request GET NAME | HTTP Request            | Retrieves full intent info from Dialogflow | Keyword validation    | simulated delay         | Get the full intent here (in JSON)                                                                                      |
| simulated delay       | Wait                    | Waits 1 second before deletion | HTTP Request GET NAME | HTTP Request ELIMINAR   | After getting the intent name, we wait 1 second before deleting it.                                                    |
| HTTP Request ELIMINAR | HTTP Request            | Deletes the intent via Dialogflow API | simulated delay       | Confirmation message    | The intent is removed                                                                                                   |
| Confirmation message  | Telegram message        | Sends deletion confirmation to user | HTTP Request ELIMINAR | None                   | The intent is removed. Message details confirmation.                                                                    |
| Sticky Note           | Sticky Note             | Instructional note             | None                  | None                   | ## Get the full intent. Here you get the entire content of the intent (in JSON)                                         |
| Sticky Note1          | Sticky Note             | Instructional note             | None                  | None                   | ## The intent is removed. Confirmation message details.                                                                 |
| Sticky Note2          | Sticky Note             | Instructional note             | None                  | None                   | ## The intent is removed                                                                                                 |
| Sticky Note3          | Sticky Note             | Instructional note             | None                  | None                   | ## The user and the submitted keyword are validated.                                                                    |
| Sticky Note4          | Sticky Note             | Instructional note             | None                  | None                   | Get the intent name to use in the confirmation message                                                                   |
| Sticky Note5          | Sticky Note             | Instructional note             | None                  | None                   | Instructions on how to extract the Intent ID from Dialogflow URL                                                         |
| Sticky Note6          | Sticky Note             | Instructional note             | None                  | None                   | After getting the intent name, we wait 1 second before deleting it.                                                     |
| HTTP Request GET      | HTTP Request            | (Unused node) Possibly test or legacy | None                  | None                   |                                                                                                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger node**  
   - Type: telegramTrigger  
   - Parameters: Listen for "message" updates only  
   - Webhook ID: auto-generated  
   - Purpose: Start workflow on incoming Telegram messages

2. **Add IF node "User validation by ID"**  
   - Type: IF  
   - Condition: Check if `{{$json.message.from.id}}` equals your authorized Telegram user ID (replace `Telegram_user_ID` with actual ID)  
   - Input: Telegram Trigger  
   - True output: Proceed; False output: Send invalid user message

3. **Add Telegram node "Invalid user message"**  
   - Type: telegram  
   - Parameters:  
     - Text: "Invalid user\nUsuario inválido"  
     - Chat ID: `{{$json.message.from.id}}` from Telegram Trigger  
   - Triggered on User validation by ID false branch

4. **Add IF node "Keyword validation"**  
   - Type: IF  
   - Condition: Check if message text equals string "delete" exactly (case sensitive)  
   - Input: User validation by ID true branch  
   - True output: Proceed; False output: Send invalid keyword message

5. **Add Telegram node "Invalid keyword message"**  
   - Type: telegram  
   - Parameters:  
     - Text: "Invalid word\nPalabra inválida"  
     - Chat ID: `{{$json.message.from.id}}`  
   - Triggered on Keyword validation false branch

6. **Add HTTP Request "HTTP Request GET NAME"**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://dialogflow.googleapis.com/v2/projects/{PROJECT_ID}/agent/intents/{INTENT_ID}` replacing placeholders with your Dialogflow project and intent IDs  
   - Query Parameters: `intentView=INTENT_VIEW_FULL`  
   - Authentication: Google API OAuth2 credentials configured with appropriate scopes for Dialogflow  
   - Input: Keyword validation true branch

7. **Add Wait node "simulated delay"**  
   - Type: Wait  
   - Duration: 1 second  
   - Input: HTTP Request GET NAME output

8. **Add HTTP Request "HTTP Request ELIMINAR"**  
   - Type: HTTP Request  
   - Method: DELETE  
   - URL: `https://dialogflow.googleapis.com/v2/projects/{PROJECT_ID}/agent/intents/{INTENT_ID}` (same as GET, with your project and intent IDs)  
   - Authentication: Google API OAuth2 credentials  
   - Input: simulated delay output

9. **Add Telegram node "Confirmation message"**  
   - Type: telegram  
   - Text:  
     ```
     =The intent: *{{$('HTTP Request GET NAME').item.json.displayName}}* has been removed.
     
     El intent: *{{$('HTTP Request GET NAME').item.json.displayName}}* ha sido eliminado.
     ```  
   - Chat ID: `{{$('Telegram Trigger').item.json.message.from.id}}`  
   - Execute Once: true  
   - Input: HTTP Request ELIMINAR output

10. **Connect nodes in this order:**  
    Telegram Trigger → User validation by ID →  
    - If false → Invalid user message  
    - If true → Keyword validation →  
      - If false → Invalid keyword message  
      - If true → HTTP Request GET NAME → simulated delay → HTTP Request ELIMINAR → Confirmation message

11. **Configure credentials:**  
    - Set up Telegram credentials with your bot token  
    - Set up Google API OAuth2 credentials with scopes to access and delete Dialogflow intents (e.g., `https://www.googleapis.com/auth/dialogflow`)  
    - Assign credentials to HTTP Request nodes accordingly

12. **Replace placeholder values:**  
    - In HTTP Request URLs replace `{PROJECT_ID}` with your Google Cloud project ID  
    - Replace `{INTENT_ID}` with the ID of the intent to be deleted (extracted as described in Sticky Note5)  
    - In User validation by ID IF node, replace `Telegram_user_ID` with your authorized Telegram user ID

13. **Optional:** Add Sticky Note nodes at relevant points to provide context and instructions for maintainers.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                          | Context or Link                                                                                                   |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------|
| 🔍 Get the **Intent ID** from the URL in Dialogflow: The intent ID is the last part of the Dialogflow intent URL before `/edit`. Example: `https://dialogflow.cloud.google.com/#/agent/{PROJECT_ID}/intents/{INTENT_ID}/edit`. Use this intent ID when configuring the URLs for GET and DELETE requests in this workflow.                                                           | See Sticky Note5 content in the workflow                                                                         |
| The workflow requires a Google Cloud project with Dialogflow API enabled and OAuth2 credentials configured for API access. The Telegram bot token must be set up with permissions to receive messages and send responses.                                                                                                                                               | Credential setup instructions for Google API and Telegram Bot                                                    |
| The workflow is designed to respond only to a specific authorized Telegram user ID for security reasons. This prevents unauthorized intent deletions.                                                                                                                                                                                                             | Sticky Note3 and node "User validation by ID"                                                                    |
| The 1-second delay after fetching the intent name before deletion ensures the API calls are sequential and reduces risk of race conditions or rate limits.                                                                                                                                                                                                           | Sticky Note6                                                                                                     |
| Confirmation messages are sent only once per successful deletion to avoid spamming.                                                                                                                                                                                                                                                                                 | Telegram node "Confirmation message" configured with `executeOnce`                                              |
| Workflow is inactive by default (`"active": false`), so enable it in n8n for operation.                                                                                                                                                                                                                                                                              | Workflow metadata                                                                                                 |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All processed data is legal and public.