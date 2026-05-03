Automated Facebook Message Response with GPT-4o and Airtable Inventory Matching

https://n8nworkflows.xyz/workflows/automated-facebook-message-response-with-gpt-4o-and-airtable-inventory-matching-11321


# Automated Facebook Message Response with GPT-4o and Airtable Inventory Matching

### 1. Workflow Overview

This workflow automates customer support for Facebook Messenger by integrating AI-driven message understanding with product inventory data from Airtable. It processes incoming Facebook Direct Messages (DMs) to identify product-related queries, matches these against a live inventory, and sends automated replies to customers accordingly.

Logical blocks:

- **1.1 Scheduled Trigger & Facebook Message Retrieval:** Periodically fetches recent Facebook conversations and extracts the latest message in each.
- **1.2 Message Validation & Logging:** Validates message structure and logs invalid messages for auditing.
- **1.3 AI Intent Extraction:** Uses GPT-4o (Azure OpenAI) to parse the customer's message, extracting product names and intent.
- **1.4 Inventory Data Fetching:** Retrieves the full product inventory from Airtable for matching.
- **1.5 AI-Powered Inventory Matching:** Combines AI-extracted data with inventory and uses AI again to determine product availability and generate reply text.
- **1.6 Automated Facebook Reply:** Sends the generated reply back to the customer on Facebook Messenger based on product availability.
- **1.7 Error Handling & Logging:** Logs improperly structured messages to Google Sheets for review.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Scheduled Trigger & Facebook Message Retrieval

**Overview:**  
This block triggers the workflow every hour to fetch the latest Facebook conversations and their messages.

**Nodes Involved:**  
- Trigger – Fetch New Facebook Messages (Every Hour)  
- Fetch Facebook Conversation List  
- Fetch Facebook Conversation Messages  
- Extract Latest Facebook Message  

**Node Details:**

- **Trigger – Fetch New Facebook Messages (Every Hour)**  
  - *Type:* Schedule Trigger  
  - *Role:* Starts the workflow every hour automatically.  
  - *Configuration:* Interval set to 1 hour.  
  - *Inputs:* None (trigger node).  
  - *Outputs:* Initiates Facebook conversation fetching.  
  - *Potential Failures:* Schedule misconfiguration, node downtime.

- **Fetch Facebook Conversation List**  
  - *Type:* Facebook Graph API  
  - *Role:* Retrieves the list of active conversations from Facebook Messenger.  
  - *Configuration:* Uses Facebook Graph API v23.0, edge "conversations". Access token is passed via credentials.  
  - *Inputs:* Trigger node output.  
  - *Outputs:* List of conversation IDs.  
  - *Potential Failures:* Auth errors, API rate limits, expired tokens.

- **Fetch Facebook Conversation Messages**  
  - *Type:* Facebook Graph API  
  - *Role:* Fetches messages from the latest conversation fetched.  
  - *Configuration:* Uses conversation ID from previous node, queries "messages{message,from,to,created_time}".  
  - *Inputs:* Conversation list node output.  
  - *Outputs:* Full message list for the conversation.  
  - *Potential Failures:* Empty conversation list, API errors, permissions.

- **Extract Latest Facebook Message**  
  - *Type:* Code  
  - *Role:* Sorts messages by created time and extracts the most recent message to process.  
  - *Configuration:* JavaScript code sorts messages descending by created_time and returns the first (latest).  
  - *Inputs:* Facebook messages node output.  
  - *Outputs:* Single latest message object.  
  - *Edge Cases:* No messages available, malformed message data.

---

#### 2.2 Message Validation & Logging

**Overview:**  
Validates the structure and presence of necessary data in the extracted message. Logs invalid messages into Google Sheets for manual review.

**Nodes Involved:**  
- Validate Record Structure  
- Log Invalid Records to Google Sheet  

**Node Details:**

- **Validate Record Structure**  
  - *Type:* If  
  - *Role:* Checks if the message JSON has a non-empty `id` field to ensure valid structure.  
  - *Configuration:* Condition checks that `$json.id` is not empty.  
  - *Inputs:* Extracted latest message.  
  - *Outputs:* True branch (valid) proceeds to AI processing, false branch logs to Google Sheets.  
  - *Edge Cases:* Missing or malformed message id, false negatives on validation.

- **Log Invalid Records to Google Sheet**  
  - *Type:* Google Sheets  
  - *Role:* Appends invalid message records to a designated Google Sheet for tracking.  
  - *Configuration:* Append operation, using OAuth2 credentials. Sheet name and document ID must be configured.  
  - *Inputs:* Invalid message from validation.  
  - *Outputs:* None downstream.  
  - *Potential Failures:* Google Sheets API limits, auth errors, misconfigured sheet/document.

---

#### 2.3 AI Intent Extraction

**Overview:**  
Uses GPT-4o model via Azure OpenAI to extract the product name, customer intent, and a normalized product name from the customer’s latest message.

**Nodes Involved:**  
- AI – Extract Product & Customer Intent  
- Configure GPT-4o — Message Classification Model  

**Node Details:**

- **Configure GPT-4o — Message Classification Model**  
  - *Type:* Langchain LM Chat (Azure OpenAI)  
  - *Role:* Provides the GPT-4o model interface with Azure OpenAI credentials.  
  - *Configuration:* Model set to "gpt-4o", no special options.  
  - *Inputs:* Text prompt with customer message.  
  - *Outputs:* AI response JSON string.  
  - *Failure Modes:* API errors, model unavailability, quota exceeded.

- **AI – Extract Product & Customer Intent**  
  - *Type:* Langchain Agent  
  - *Role:* Sends prompt to GPT-4o to extract structured data from customer message.  
  - *Configuration:* Prompt instructs AI to extract product name, intent, cleaned product name, and return only JSON. System message enforces JSON-only output and flexible typo handling.  
  - *Inputs:* Latest message text.  
  - *Outputs:* JSON string with extracted fields (`query`, `product_name`, `intent`, `found: false`).  
  - *Edge Cases:* Ambiguous messages, poor AI extraction, JSON parsing errors downstream.

---

#### 2.4 Inventory Data Fetching

**Overview:**  
Fetches the entire inventory of products from Airtable to provide data for product matching.

**Nodes Involved:**  
- Fetch Inventory Records from Airtable  

**Node Details:**

- **Fetch Inventory Records from Airtable**  
  - *Type:* Airtable  
  - *Role:* Retrieves product inventory records from a configured Airtable base and table.  
  - *Configuration:* Base "Lead Manager" and table "Inventory". Authenticated via Airtable Personal Access Token. Operation is "search" (fetch all).  
  - *Inputs:* Triggered after AI extraction step.  
  - *Outputs:* List of inventory records as JSON objects.  
  - *Potential Failures:* API limits, invalid credentials, missing base/table.

---

#### 2.5 AI-Powered Inventory Matching

**Overview:**  
Merges AI-extracted customer query with inventory data, then runs a second AI prompt to determine if the product exists and prepares a human-readable reply.

**Nodes Involved:**  
- Merge AI Output With Inventory Dataset  
- Build Combined AI + Inventory Payload (Code)  
- AI – Match Requested Product in Inventory  
- Configure GPT-4o — Product Matching Model  
- Parse AI Product Match JSON  

**Node Details:**

- **Merge AI Output With Inventory Dataset**  
  - *Type:* Merge  
  - *Role:* Combines AI extraction output with inventory records for joint analysis.  
  - *Inputs:* Incoming from AI extraction and Airtable fetch nodes.  
  - *Outputs:* Combined array of AI data + inventory.  
  - *Edge Cases:* Mismatch of data length, empty inventory.

- **Build Combined AI + Inventory Payload (Code)**  
  - *Type:* Code  
  - *Role:* Parses AI JSON string into an object and packages it with inventory data into one payload.  
  - *Configuration:* JS code parses AI output JSON safely, handles parse errors, and consolidates inventory records.  
  - *Inputs:* Merged data from previous node.  
  - *Outputs:* Single JSON object containing AI and inventory data.  
  - *Edge Cases:* AI output parsing failure, empty inventory array.

- **Configure GPT-4o — Product Matching Model**  
  - *Type:* Langchain LM Chat (Azure OpenAI)  
  - *Role:* Provides GPT-4o model interface for product matching prompt.  
  - *Configuration:* Same as AI extraction model but used for matching step.  
  - *Inputs:* Combined data payload prompt.  
  - *Outputs:* AI matching JSON response string.  
  - *Failure Modes:* API errors, token limits.

- **AI – Match Requested Product in Inventory**  
  - *Type:* Langchain Agent  
  - *Role:* Runs AI prompt to check if product exists in inventory, returns match details and reply text as JSON.  
  - *Configuration:* Prompt includes customer query, intent, inventory list, rules for matching, and strict JSON-only response.  
  - *Inputs:* Combined AI + inventory payload.  
  - *Outputs:* JSON string with matching result: product_name, found (bool), matched_item object, reply text, confidence score.  
  - *Edge Cases:* Ambiguous matches, false positives/negatives, JSON parse errors downstream.

- **Parse AI Product Match JSON**  
  - *Type:* Code  
  - *Role:* Parses AI matching JSON string into usable JSON object for decision-making.  
  - *Configuration:* JS tries JSON.parse, throws error on failure to halt workflow.  
  - *Inputs:* Raw AI output string from matching node.  
  - *Outputs:* Parsed JSON object with match info.  
  - *Edge Cases:* Malformed AI output JSON, parse exceptions.

---

#### 2.6 Automated Facebook Reply

**Overview:**  
Sends an automated Facebook Messenger reply to the customer based on whether the product was found or not.

**Nodes Involved:**  
- Check If Product Exists  
- Send Facebook Reply — Product Found  
- Send Facebook Reply — Product Not Found  

**Node Details:**

- **Check If Product Exists**  
  - *Type:* If  
  - *Role:* Branches workflow based on the boolean `found` field in AI match output.  
  - *Configuration:* Checks if `$json.found === true`.  
  - *Inputs:* Parsed AI product match JSON.  
  - *Outputs:* True branch if product found; false branch if not found.  
  - *Edge Cases:* Missing `found` field, incorrect boolean values.

- **Send Facebook Reply — Product Found**  
  - *Type:* Facebook Graph API  
  - *Role:* Sends a Messenger reply to the customer with product found message.  
  - *Configuration:* Uses Facebook Graph API v23.0, POST to "messages" edge. Recipient ID is extracted from latest message sender id. Message text is AI-generated reply.  
  - *Inputs:* True branch from check node.  
  - *Outputs:* None downstream.  
  - *Potential Failures:* API errors, token expiry, message send failure.

- **Send Facebook Reply — Product Not Found**  
  - *Type:* Facebook Graph API  
  - *Role:* Sends a polite “product not available” reply to the customer.  
  - *Configuration:* Same as above but triggered on false branch.  
  - *Inputs:* False branch from check node.  
  - *Outputs:* None downstream.  
  - *Potential Failures:* Same as above.

---

#### 2.7 Error Handling & Logging (Implicit)

- Invalid messages detected in validation are logged into Google Sheets for manual review.
- AI JSON parsing errors throw exceptions, halting workflow and signaling need for intervention.
- Facebook API or Airtable failures may cause workflow retries or error notifications depending on n8n setup.

---

### 3. Summary Table

| Node Name                                  | Node Type                          | Functional Role                                | Input Node(s)                        | Output Node(s)                             | Sticky Note                                                                                                      |
|--------------------------------------------|----------------------------------|-----------------------------------------------|------------------------------------|--------------------------------------------|------------------------------------------------------------------------------------------------------------------|
| Trigger – Fetch New Facebook Messages (Every Hour) | Schedule Trigger                 | Starts workflow every hour                     | None                               | Fetch Facebook Conversation List            | See Sticky Note: Overall workflow summary and purpose                                                           |
| Fetch Facebook Conversation List           | Facebook Graph API                | Fetches Facebook conversation IDs             | Trigger – Fetch New Facebook Messages | Fetch Facebook Conversation Messages         | Intake & Validation — Facebook Messages                                                                          |
| Fetch Facebook Conversation Messages       | Facebook Graph API                | Retrieves messages from conversation           | Fetch Facebook Conversation List   | Validate Record Structure                    | Intake & Validation — Facebook Messages                                                                          |
| Validate Record Structure                   | If                              | Checks message id exists                        | Fetch Facebook Conversation Messages | Extract Latest Facebook Message / Log Invalid Records to Google Sheet | Intake & Validation — Facebook Messages                                                                          |
| Log Invalid Records to Google Sheet         | Google Sheets                    | Logs invalid messages                           | Validate Record Structure (false)  | None                                       | Intake & Validation — Facebook Messages                                                                          |
| Extract Latest Facebook Message             | Code                            | Extracts most recent message                    | Validate Record Structure (true)   | AI – Extract Product & Customer Intent / Fetch Inventory Records from Airtable | Intake & Validation — Facebook Messages                                                                          |
| AI – Extract Product & Customer Intent      | Langchain Agent                  | Parses message to extract product and intent  | Extract Latest Facebook Message    | Merge AI Output With Inventory Dataset       | AI Intent Understanding — Message Analysis                                                                       |
| Configure GPT-4o — Message Classification Model | Langchain LM Chat (Azure OpenAI) | Sets GPT-4o model for intent extraction        | AI – Extract Product & Customer Intent (AI model) | AI – Extract Product & Customer Intent (response) | AI Intent Understanding — Message Analysis                                                                       |
| Fetch Inventory Records from Airtable       | Airtable                        | Pulls product inventory data                    | AI – Extract Product & Customer Intent | Merge AI Output With Inventory Dataset       | Inventory Matching — AI + Airtable Dataset                                                                       |
| Merge AI Output With Inventory Dataset      | Merge                          | Combines AI extraction and inventory data      | AI – Extract Product & Customer Intent, Fetch Inventory Records from Airtable | Build Combined AI + Inventory Payload (                  | Inventory Matching — AI + Airtable Dataset                                                                       |
| Build Combined AI + Inventory Payload (     | Code                            | Parses AI JSON and packages with inventory     | Merge AI Output With Inventory Dataset | AI – Match Requested Product in Inventory     | Inventory Matching — AI + Airtable Dataset                                                                       |
| AI – Match Requested Product in Inventory   | Langchain Agent                  | AI checks product availability and creates reply | Build Combined AI + Inventory Payload | Parse AI Product Match JSON                   | Inventory Matching — AI + Airtable Dataset                                                                       |
| Configure GPT-4o — Product Matching Model   | Langchain LM Chat (Azure OpenAI) | Sets GPT-4o model for product matching          | AI – Match Requested Product in Inventory (AI model) | AI – Match Requested Product in Inventory (response) | Inventory Matching — AI + Airtable Dataset                                                                       |
| Parse AI Product Match JSON                  | Code                            | Parses AI matching JSON string                   | AI – Match Requested Product in Inventory | Check If Product Exists                      | Inventory Matching — AI + Airtable Dataset                                                                       |
| Check If Product Exists                      | If                              | Branches on product found or not                 | Parse AI Product Match JSON        | Send Facebook Reply — Product Found / Send Facebook Reply — Product Not Found | Auto-Reply Delivery — Facebook Messenger                                                                         |
| Send Facebook Reply — Product Found          | Facebook Graph API               | Sends product found reply to customer           | Check If Product Exists (true)     | None                                       | Auto-Reply Delivery — Facebook Messenger                                                                         |
| Send Facebook Reply — Product Not Found      | Facebook Graph API               | Sends product not found reply to customer       | Check If Product Exists (false)    | None                                       | Auto-Reply Delivery — Facebook Messenger                                                                         |
| Sticky Note                                  | Sticky Note                     | Workflow summary and overview                    | None                               | None                                       | See sticky note content                                                                                           |
| Sticky Note1                                 | Sticky Note                     | Intake & validation summary                       | None                               | None                                       | Intake & Validation — Facebook Messages                                                                          |
| Sticky Note2                                 | Sticky Note                     | AI Intent understanding summary                   | None                               | None                                       | AI Intent Understanding — Message Analysis                                                                       |
| Sticky Note3                                 | Sticky Note                     | Inventory matching summary                         | None                               | None                                       | Inventory Matching — AI + Airtable Dataset                                                                       |
| Sticky Note4                                 | Sticky Note                     | Auto-reply delivery summary                         | None                               | None                                       | Auto-Reply Delivery — Facebook Messenger                                                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Set interval to 1 hour.  
   - This triggers the workflow periodically.

2. **Add a Facebook Graph API Node to Fetch Conversation List**  
   - Type: Facebook Graph API  
   - Operation: GET  
   - Edge: "conversations"  
   - API version: v23.0  
   - Configure OAuth2 credentials for Facebook Graph API.  
   - Connect Schedule Trigger output to this node.

3. **Add a Facebook Graph API Node to Fetch Conversation Messages**  
   - Type: Facebook Graph API  
   - Operation: GET  
   - Node ID: Use expression `={{ $json.data[0].id }}` from previous node to fetch latest conversation ID.  
   - Query parameters:  
     - access_token (from credentials)  
     - fields: "messages{message,from,to,created_time}"  
   - Connect previous node output to this node.

4. **Add an If Node to Validate Message Structure**  
   - Type: If  
   - Condition: Check if `$json.id` is not empty (string not empty).  
   - Connect Facebook messages node output to this node.

5. **Add a Google Sheets Node for Logging Invalid Messages**  
   - Type: Google Sheets  
   - Operation: Append  
   - Configure OAuth2 credentials with access to your Google Sheet.  
   - Set document ID and sheet name for logs.  
   - Connect If node's false branch to this node.

6. **Add a Code Node to Extract Latest Facebook Message**  
   - Type: Code  
   - JavaScript code:  
     ```js
     const msgs = $json.messages.data;
     msgs.sort((a, b) => new Date(b.created_time) - new Date(a.created_time));
     return [{ json: msgs[0] }];
     ```  
   - Connect If node's true branch to this node.

7. **Configure GPT-4o Model Node for Intent Extraction**  
   - Type: Langchain LM Chat (Azure OpenAI)  
   - Model: gpt-4o  
   - Connect to Azure OpenAI credentials.  
   - Connect Code node output to this node's AI model input.

8. **Add Langchain Agent Node for AI Intent Extraction**  
   - Type: Langchain Agent  
   - Prompt: Use the provided prompt to extract product, intent, and cleaned product name from message text.  
   - Set system message to enforce JSON only output and flexible name detection.  
   - Input text: `"You received this customer message:\n\n\"{{ $json.message }}\"\n\nExtract the following:\n1. The product the customer is asking for.\n2. The intent (ex: availability, price inquiry, general question).\n3. The cleaned product name (lowercase, no extra words).\n4. Return ONLY a JSON object.\n\nExample format to return:\n{\n  \"query\": \"...\",\n  \"product_name\": \"...\",\n  \"intent\": \"...\",\n  \"found\": false\n}"`  
   - Connect Code node output to this node.

9. **Add Airtable Node to Fetch Inventory**  
   - Type: Airtable  
   - Operation: Search (fetch all records)  
   - Base: Your Airtable base containing inventory  
   - Table: Inventory table  
   - Authenticate with Airtable Personal Access Token credentials.  
   - Connect AI intent extraction node output to this node.

10. **Add Merge Node**  
    - Type: Merge  
    - Mode: Default (append)  
    - Connect AI intent extraction node output to first input.  
    - Connect Airtable inventory node output to second input.

11. **Add Code Node to Build Combined Payload**  
    - Type: Code  
    - JS code to parse AI output JSON and combine with inventory records into single JSON object.  
    - Connect Merge node output to this node.

12. **Configure GPT-4o Model Node for Product Matching**  
    - Type: Langchain LM Chat (Azure OpenAI)  
    - Model: gpt-4o  
    - Connect Azure OpenAI credentials.  
    - Connect Code node output to this node's AI model input.

13. **Add Langchain Agent Node for Inventory Product Matching**  
    - Type: Langchain Agent  
    - Prompt: Provide combined customer query and inventory list, instruct AI to determine product existence, return JSON with product details and reply message.  
    - System message enforces JSON-only output and matching rules.  
    - Connect combined payload node output to this node.

14. **Add Code Node to Parse AI Product Match JSON**  
    - Type: Code  
    - JS code to safely parse AI output string into JSON object.  
    - Connect AI matching agent output to this node.

15. **Add If Node to Check Product Existence**  
    - Type: If  
    - Condition: Check if `$json.found === true`.  
    - Connect parsing code node output to this node.

16. **Add Facebook Graph API Node to Send Reply if Product Found**  
    - Type: Facebook Graph API  
    - Operation: POST  
    - Edge: "messages"  
    - Recipient ID: Extract from latest message sender id expression `={{ JSON.stringify({ id: $('Extract Latest Facebook Message').first().json.from.id }) }}`  
    - Message: Use AI-generated reply from `$json.reply`.  
    - Connect If node true branch to this node.  
    - Use appropriate Facebook Graph API OAuth2 credentials.

17. **Add Facebook Graph API Node to Send Reply if Product Not Found**  
    - Same as above but connected to If node false branch.

18. **Optionally Add Sticky Notes**  
    - Add descriptive sticky notes for documentation and clarity in n8n editor.

---

### 5. General Notes & Resources

| Note Content                                                                                                                  | Context or Link                                                                                          |
|-------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------|
| This workflow enables instant, automated customer support on Facebook Messenger by combining AI intent extraction and inventory matching for rapid replies. | Sticky Note summarizing workflow purpose.                                                               |
| Uses Azure OpenAI GPT-4o for natural language understanding and product matching tasks.                                         | Requires Azure OpenAI API credentials configured in n8n.                                               |
| Airtable serves as the live product catalog for inventory reference.                                                           | Airtable Personal Access Token with correct base and table access required.                             |
| Google Sheets logs invalid or malformed messages for review and quality control.                                               | Requires Google Sheets OAuth2 credentials.                                                             |
| Facebook Graph API v23.0 is used to fetch conversations and send messages, requiring appropriate Facebook app permissions.    | Ensure Facebook app has required Messenger permissions and tokens refreshed regularly.                  |
| AI prompts are designed to return strict JSON outputs to facilitate automated parsing and decision-making.                     | Any deviation from strict JSON will cause parsing errors and halt workflow.                             |
| Potential failure points include API limits, token expirations, malformed AI outputs, and message structure edge cases.       | Monitor workflow executions and handle errors with alerts or retries as needed.                         |

---

**Disclaimer:** The provided text is exclusively derived from an automated workflow built with n8n, an integration and automation tool. This processing strictly adheres to valid content policies and contains no illegal, offensive, or protected material. All data handled is legal and publicly accessible.