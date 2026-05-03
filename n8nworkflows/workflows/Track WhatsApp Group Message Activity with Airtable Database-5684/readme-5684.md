Track WhatsApp Group Message Activity with Airtable Database

https://n8nworkflows.xyz/workflows/track-whatsapp-group-message-activity-with-airtable-database-5684


# Track WhatsApp Group Message Activity with Airtable Database

### 1. Workflow Overview

This workflow is designed to track user engagement within a specific WhatsApp group by counting message interactions and updating an Airtable database accordingly. It supports multiple message types—including text, emoji reactions, voice messages, and images—and increments a per-user message count stored in Airtable. The workflow is ideal for use cases such as SaaS app engagement monitoring, weekly raffles, or gamification systems where tracking user participation is essential.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Receives incoming WhatsApp messages via a webhook.
- **1.2 Group Filtering:** Checks if the message originates from the targeted WhatsApp group.
- **1.3 Message Type Identification:** Determines the type of message (text, emoji reaction, voice, image).
- **1.4 User Lookup:** Searches the Airtable database for the user record based on WhatsApp ID.
- **1.5 Engagement Counting:** Increments the user's message count.
- **1.6 Database Update:** Updates the Airtable record with the new count and last interaction date.
- **1.7 Workflow End:** Optionally, terminates processing for irrelevant messages.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  Captures incoming WhatsApp messages sent by Whapi (WhatsApp API) through a webhook.

- **Nodes Involved:**  
  - `Webhook`

- **Node Details:**  
  - **Webhook**  
    - Type: Webhook  
    - Role: Entry point for incoming HTTP POST requests containing WhatsApp message data.  
    - Configuration:  
      - HTTP Method: POST  
      - Path: Unique webhook path (`bf830497-e200-47da-a10e-8f710cd3c9f6`)  
    - Inputs: HTTP requests from Whapi  
    - Outputs: Raw JSON message payload forwarded downstream  
    - Edge Cases:  
      - Unauthorized or malformed requests may cause missing or invalid data.  
      - Webhook path must be secured to prevent unauthorized access.  
    - Sticky Note:  
      - "Webhook from Whapi: Catch the webhook from Whapi when the message is received"

#### 2.2 Group Filtering

- **Overview:**  
  Checks if the incoming message is from the target WhatsApp group by comparing the `chat_id` to a predefined group ID.

- **Nodes Involved:**  
  - `Nachricht?` (IF node)  
  - `No Operation, do nothing`

- **Node Details:**  
  - **Nachricht? (IF)**  
    - Type: IF  
    - Role: Filters messages by group ID  
    - Configuration:  
      - Condition: Checks if `body.messages[0].chat_id` equals `"120363417847196227@g.us"` (the target WhatsApp group ID)  
      - Case sensitive, strict type validation  
    - Inputs: Output from `Webhook` node  
    - Outputs:  
      - `True` branch if message is from the target group  
      - `False` branch otherwise  
    - Edge Cases:  
      - Messages without `chat_id` or malformed JSON may cause condition failure.  
      - If the group ID changes, the condition must be updated.  
    - Sticky Note:  
      - "Right group? Checks if the message was sent in the right group."

  - **No Operation, do nothing**  
    - Type: NoOp  
    - Role: Terminates processing for messages not from the target group  
    - Inputs: `False` branch from `Nachricht?`  
    - Outputs: None

#### 2.3 Message Type Identification

- **Overview:**  
  Determines the type of WhatsApp message received to ensure the workflow processes only relevant message types.

- **Nodes Involved:**  
  - `Switch`

- **Node Details:**  
  - **Switch**  
    - Type: Switch  
    - Role: Routes the workflow based on message content type  
    - Configuration: Four outputs dependent on conditions:  
      - **Text:** Checks if `body.messages[0].text.body` exists  
      - **Emoji reaction:** Checks if `body.messages[0].action.emoji` exists  
      - **Voice:** Checks if `body.messages[0].type` equals `"voice"`  
      - **Image:** Checks if `body.messages[0].type` equals `"image"`  
    - Inputs: `True` branch from `Nachricht?`  
    - Outputs: All four outputs proceed to the same next node (`Suche nach WA_ID`)  
    - Edge Cases:  
      - Messages without any of these types will not be routed forward.  
      - New message types require adding additional cases.  
    - Sticky Note:  
      - "Text, emoji, voice, image? Check if some of these messages are the ones mentioned."

#### 2.4 User Lookup

- **Overview:**  
  Searches the Airtable database to find if the user (identified by WhatsApp ID) already exists in the engagement tracking table.

- **Nodes Involved:**  
  - `Suche nach WA_ID`

- **Node Details:**  
  - **Suche nach WA_ID (Airtable)**  
    - Type: Airtable node  
    - Role: Performs a search operation in Airtable to find a record matching the WhatsApp ID of the message sender  
    - Configuration:  
      - Base: WhatsApp Engagement Database (`appREiqyOxTYwsigc`)  
      - Table: Table 1 (`tblIf7YbtyvUvDNm0`)  
      - Operation: Search with filter formula `{WhatsApp_ID} = {{ $json.body.messages[0].from }}`  
      - Always outputs data to allow downstream processing  
    - Inputs: All outputs from `Switch` node  
    - Outputs: The found record(s) with engagement data or empty if none found  
    - Edge Cases:  
      - If no record is found, downstream nodes must handle null or empty data.  
      - Airtable API rate limits or authentication issues may cause failures.  
    - Sticky Note:  
      - "Search for user: Look up the right Airtable record via WhatsApp ID"

#### 2.5 Engagement Counting

- **Overview:**  
  Retrieves the current message count for the user and increments it by one.

- **Nodes Involved:**  
  - `+1`

- **Node Details:**  
  - **+1 (Code)**  
    - Type: Code (JavaScript)  
    - Role: Increments the current engagement count  
    - Configuration:  
      - Retrieves the current `Count` from the first input item JSON  
      - Adds 1 to the count  
      - Returns the updated `Count`  
      - Code snippet:  
        ```js
        var count = $input.first().json.Count; // Get current count
        count += 1; // Increment by 1
        return { Count: count }; // Return updated count
        ```  
    - Inputs: Output from `Suche nach WA_ID`  
    - Outputs: JSON with updated `Count`  
    - Edge Cases:  
      - If `Count` is undefined or null, may cause errors or NaN results. A fallback to zero is recommended in such cases but is not implemented here.  
      - Expression failures due to missing fields.  
    - Sticky Note:  
      - "Existing User (+1): Adds a +1 to the message count"

#### 2.6 Database Update

- **Overview:**  
  Updates the user's record in Airtable with the new message count and the date of the last interaction.

- **Nodes Involved:**  
  - `Airtable Update`

- **Node Details:**  
  - **Airtable Update**  
    - Type: Airtable node  
    - Role: Updates the engagement count and last interaction date in Airtable for the user  
    - Configuration:  
      - Base: WhatsApp Engagement Database (`appREiqyOxTYwsigc`)  
      - Table: Table 1 (`tblIf7YbtyvUvDNm0`)  
      - Operation: Update  
      - Matching Column: `WhatsApp_ID`  
      - Fields updated:  
        - `Count` set to the incremented count from `+1` node  
        - `WhatsApp_ID` set from webhook message sender  
        - `Last interaction` set to current date in `yyyy.MM.dd` format  
      - Does not attempt type conversion or string conversion  
    - Inputs: Output from `+1` node  
    - Outputs: Updated record confirmation (not used further)  
    - Edge Cases:  
      - If the matching record is not found, update will fail silently or cause errors.  
      - Airtable API limits, authentication errors, or network issues may cause failures.  
    - Sticky Note:  
      - (No specific sticky note on this node)

---

### 3. Summary Table

| Node Name             | Node Type       | Functional Role                     | Input Node(s)        | Output Node(s)         | Sticky Note                                                                                   |
|-----------------------|-----------------|-----------------------------------|----------------------|------------------------|-----------------------------------------------------------------------------------------------|
| Webhook               | Webhook         | Receive incoming WhatsApp webhook | —                    | Nachricht?             | Webhook from Whapi: Catch the webhook from Whapi when the message is received                 |
| Nachricht?            | IF              | Filter messages by WhatsApp group | Webhook              | Switch, No Operation    | Right group? Checks if the message was sent in the right group.                              |
| No Operation, do nothing | NoOp           | End processing for irrelevant msgs| Nachricht? (False)   | —                      |                                                                                               |
| Switch                | Switch          | Identify message type             | Nachricht? (True)     | Suche nach WA_ID (4x)  | Text, emoji, voice, image? Check if some of these messages are the ones mentioned.            |
| Suche nach WA_ID       | Airtable        | Lookup user record by WhatsApp ID | Switch (all outputs)  | +1                     | Search for user: Look up the right Airtable record via WhatsApp ID                            |
| +1                    | Code            | Increment message count           | Suche nach WA_ID      | Airtable Update        | Existing User (+1): Adds a +1 to the message count                                           |
| Airtable Update       | Airtable        | Update engagement count and date  | +1                    | —                      |                                                                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Webhook node:**  
   - Type: Webhook  
   - Parameters:  
     - HTTP Method: POST  
     - Path: `bf830497-e200-47da-a10e-8f710cd3c9f6` (unique identifier)  
   - Purpose: Receive WhatsApp messages from Whapi API.

2. **Add an IF node named "Nachricht?":**  
   - Connect the Webhook node output to this node's input.  
   - Set condition:  
     - Expression: `{{$json.body.messages[0].chat_id}}`  
     - Operator: equals  
     - Value: `"120363417847196227@g.us"` (target WhatsApp group ID)  
   - Configure two outputs:  
     - True: proceed to message type identification  
     - False: terminate workflow.

3. **Add a No Operation node ("No Operation, do nothing"):**  
   - Connect the False output of "Nachricht?" to this node.  
   - No configuration needed; serves to end processing for non-target group messages.

4. **Add a Switch node named "Switch":**  
   - Connect the True output of "Nachricht?" to this node.  
   - Add four rules for outputs:  
     - **Text:** Check if `{{$json.body.messages[0].text.body}}` exists (string exists)  
     - **Emoji reaction:** Check if `{{$json.body.messages[0].action.emoji}}` exists  
     - **Voice:** Check if `{{$json.body.messages[0].type}}` equals `"voice"`  
     - **Image:** Check if `{{$json.body.messages[0].type}}` equals `"image"`  

5. **Add an Airtable node "Suche nach WA_ID":**  
   - Connect all four outputs of the Switch node to this node's input.  
   - Configure Airtable credentials and base:  
     - Base ID: `appREiqyOxTYwsigc` (WhatsApp Engagement Database)  
     - Table ID: `tblIf7YbtyvUvDNm0` (Table 1)  
   - Operation: Search  
   - Filter formula:  
     ```  
     {WhatsApp_ID} = {{ $json.body.messages[0].from }}  
     ```  
   - Enable "Always Output Data" to ensure the node outputs even if no record is found.

6. **Add a Code node "+1":**  
   - Connect the output of "Suche nach WA_ID" to this node.  
   - Enter JavaScript code:  
     ```js
     var count = $input.first().json.Count;
     count += 1;
     return { Count: count };
     ```  
   - This increments the current count by one.

7. **Add an Airtable node "Airtable Update":**  
   - Connect the output of the Code node to this node.  
   - Configure Airtable credentials and base/table as before.  
   - Operation: Update  
   - Matching Column: `WhatsApp_ID`  
   - Fields to update:  
     - `Count`: `={{ $json.Count }}` (from the Code node)  
     - `WhatsApp_ID`: `={{ $('Webhook').item.json.body.messages[0].from }}` (from the webhook data)  
     - `Last interaction`: `={{ $now.format('yyyy.MM.dd') }}` (current date)  
   - Mapping Mode: Define below (explicit field mapping)  
   - Do not enable type conversion or string conversion.

---

### 5. General Notes & Resources

| Note Content                                                                                                             | Context or Link                                                                                                             |
|--------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------|
| Summary of the Workflow: Tracks and encourages engagement in a WhatsApp group; ideal for SaaS apps, raffles, gamification | Sticky Note near workflow overview node: Detailed description of workflow purpose and benefits                              |
| Workflow is adaptable to new message types, groups, or reward systems                                                     | Workflow design allows adding new cases in the Switch node and updating Airtable accordingly                                 |

---

**Disclaimer:** The provided content originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and publicly available.