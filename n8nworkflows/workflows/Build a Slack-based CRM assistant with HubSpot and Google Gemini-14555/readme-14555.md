Build a Slack-based CRM assistant with HubSpot and Google Gemini

https://n8nworkflows.xyz/workflows/build-a-slack-based-crm-assistant-with-hubspot-and-google-gemini-14555


# Build a Slack-based CRM assistant with HubSpot and Google Gemini

# 1. Workflow Overview

This workflow builds a Slack-based CRM assistant that listens for Slack mentions, extracts the user’s query, searches HubSpot data across deals, companies, and contacts, then uses Google Gemini to generate a readable response and posts that response back into Slack.

Typical use cases:
- Ask for a deal by name from Slack
- Search for a contact by first name
- Search for a company using an identifier
- Return a summarized CRM answer without opening HubSpot

## 1.1 Input Reception and Query Cleanup

The workflow starts when the Slack app is mentioned in a workspace channel. Because Slack mentions are often delivered with internal formatting such as `<@USERID>`, the workflow removes angle-bracketed Slack tokens and keeps only the cleaned user query.

## 1.2 HubSpot Data Retrieval

After cleaning the Slack text, the workflow queries HubSpot in parallel:
- Deals
- Companies
- Contacts

Each branch retrieves data using HubSpot nodes and then applies a filter based on the cleaned Slack query.

## 1.3 Result Consolidation and AI Formatting

The filtered outputs from all three HubSpot branches are merged into a single stream. That merged data is passed to an AI Agent connected to a Google Gemini Chat Model, which generates the final natural-language response.

## 1.4 Slack Response Delivery

The AI-generated text is sent to a configured Slack channel using a Slack node.

---

# 2. Block-by-Block Analysis

## Block 1 — Trigger and Query Cleanup

### Overview
This block receives the Slack mention event and transforms the incoming Slack-formatted message into a clean search string. It is the entry point of the workflow and provides the query used by all downstream HubSpot filters.

### Nodes Involved
- Slack Trigger
- Code in JavaScript

### Node Details

#### 1. Slack Trigger
- **Type and technical role:** `n8n-nodes-base.slackTrigger`  
  Event source node that listens for Slack app mention events.
- **Configuration choices:**
  - Trigger type: `app_mention`
  - `watchWorkspace`: enabled
  - Uses Slack webhook/app configuration associated with the Slack app
- **Key expressions or variables used:**  
  No custom expression in this node. It emits Slack event payloads, including message text.
- **Input and output connections:**
  - Input: none
  - Output: to `Code in JavaScript`
- **Version-specific requirements:** `typeVersion: 1`
- **Edge cases or potential failure types:**
  - Slack app missing required scopes
  - Bot not invited to the target channel
  - Slack trigger webhook misconfiguration
  - Workspace event subscription not enabled properly
  - Message payload shape may vary by Slack event type
- **Sub-workflow reference:** none

#### 2. Code in JavaScript
- **Type and technical role:** `n8n-nodes-base.code`  
  Cleans Slack message text by stripping Slack-specific tokens enclosed in angle brackets.
- **Configuration choices:**
  - Runs JavaScript over all input items
  - Uses regex `/\<[^>]+\>/g` to remove any content wrapped in `< >`
  - Trims the result before returning
- **Key expressions or variables used:**
  - `item.json.text`
  - `$input.all()`
- **Input and output connections:**
  - Input: from `Slack Trigger`
  - Output:
    - to `Get many companies`
    - to `Get Contacts`
    - to `Get Deals`
- **Version-specific requirements:** `typeVersion: 2`
- **Edge cases or potential failure types:**
  - If `text` is missing, the code safely skips replacement
  - Regex removes all angle-bracket content, which may also remove useful Slack-formatted links or special entities
  - If the cleaned message becomes empty, downstream filters may return no useful matches or potentially broad matches depending on data
- **Sub-workflow reference:** none

---

## Block 2 — CRM Retrieval: Deals

### Overview
This branch searches HubSpot deals and filters the returned deal records using the cleaned Slack query. It is intended for natural-language matching against deal names.

### Nodes Involved
- Get Deals
- Filter Deals

### Node Details

#### 3. Get Deals
- **Type and technical role:** `n8n-nodes-base.hubspot`  
  Queries HubSpot deal data.
- **Configuration choices:**
  - Resource: `deal`
  - Operation: `search`
  - Authentication: app token
- **Key expressions or variables used:**  
  No visible dynamic search expression is configured in the node itself.
- **Input and output connections:**
  - Input: from `Code in JavaScript`
  - Output: to `Filter Deals`
- **Version-specific requirements:** `typeVersion: 2.2`
- **Edge cases or potential failure types:**
  - Invalid or expired HubSpot app token
  - HubSpot API rate limits
  - Search operation may not return the expected fields if defaults differ by HubSpot API version
  - If no search criteria are defined, actual behavior may depend on node defaults and HubSpot API support
- **Sub-workflow reference:** none

#### 4. Filter Deals
- **Type and technical role:** `n8n-nodes-base.filter`  
  Keeps only deal items whose `properties.dealname` contains the cleaned Slack query.
- **Configuration choices:**
  - Case-insensitive matching enabled
  - Condition type: string contains
  - Left value: `{{ $json.properties.dealname }}`
  - Right value: `{{ $('Code in JavaScript').item.json.text }}`
  - `onError`: continue regular output
- **Key expressions or variables used:**
  - `$json.properties.dealname`
  - `$('Code in JavaScript').item.json.text`
- **Input and output connections:**
  - Input: from `Get Deals`
  - Output: to `Merge` input 0
- **Version-specific requirements:** `typeVersion: 2.3`
- **Edge cases or potential failure types:**
  - Missing `properties.dealname`
  - Query text empty or undefined
  - Contains matching may be too broad for short queries
  - Because errors continue, malformed items may silently pass without stopping the workflow
- **Sub-workflow reference:** none

---

## Block 3 — CRM Retrieval: Companies

### Overview
This branch retrieves HubSpot companies and filters them against the cleaned Slack query. In the current design, the company filter compares a numeric company identifier with the Slack text.

### Nodes Involved
- Get many companies
- Filter Companies

### Node Details

#### 5. Get many companies
- **Type and technical role:** `n8n-nodes-base.hubspot`  
  Retrieves all HubSpot companies.
- **Configuration choices:**
  - Resource: `company`
  - Operation: `getAll`
  - `returnAll`: enabled
  - Authentication: app token
- **Key expressions or variables used:** none
- **Input and output connections:**
  - Input: from `Code in JavaScript`
  - Output: to `Filter Companies`
- **Version-specific requirements:** `typeVersion: 2.2`
- **Edge cases or potential failure types:**
  - Large company datasets may create long execution times or memory pressure
  - HubSpot pagination and rate limits
  - Invalid credentials
- **Sub-workflow reference:** none

#### 6. Filter Companies
- **Type and technical role:** `n8n-nodes-base.filter`  
  Filters company items by testing whether `companyId` equals the cleaned Slack query as a number.
- **Configuration choices:**
  - Condition type: number equals
  - Left value: `{{ $json.companyId }}`
  - Right value: `{{ $('Code in JavaScript').item.json.text }}`
  - Ignore case enabled, though not relevant to numeric comparison
  - Loose type validation explicitly disabled
  - `onError`: continue regular output
- **Key expressions or variables used:**
  - `$json.companyId`
  - `$('Code in JavaScript').item.json.text`
- **Input and output connections:**
  - Input: from `Get many companies`
  - Output: to `Merge` input 1
- **Version-specific requirements:** `typeVersion: 2.3`
- **Edge cases or potential failure types:**
  - This branch expects the Slack query to be a numeric company ID, not a company name
  - Non-numeric Slack text can cause comparison failures or no matches
  - Large result sets upstream can affect performance
  - Continued execution on error may hide data-shape issues
- **Sub-workflow reference:** none

---

## Block 4 — CRM Retrieval: Contacts

### Overview
This branch loads HubSpot contacts and keeps only those whose first name contains the cleaned Slack query. It is a lightweight contact lookup mechanism.

### Nodes Involved
- Get Contacts
- Filter Contacts

### Node Details

#### 7. Get Contacts
- **Type and technical role:** `n8n-nodes-base.hubspot`  
  Retrieves all HubSpot contacts.
- **Configuration choices:**
  - Operation: `getAll`
  - `returnAll`: enabled
  - Authentication: app token
- **Key expressions or variables used:** none
- **Input and output connections:**
  - Input: from `Code in JavaScript`
  - Output: to `Filter Contacts`
- **Version-specific requirements:** `typeVersion: 2.2`
- **Edge cases or potential failure types:**
  - Very large contact lists can impact runtime
  - HubSpot API throttling
  - Missing credentials or insufficient scopes
- **Sub-workflow reference:** none

#### 8. Filter Contacts
- **Type and technical role:** `n8n-nodes-base.filter`  
  Filters contacts whose first name contains the cleaned Slack query.
- **Configuration choices:**
  - Case-insensitive matching enabled
  - Condition type: string contains
  - Left value: `{{ $json.properties.firstname.value }}`
  - Right value: `{{ $('Code in JavaScript').item.json.text }}`
  - `onError`: continue regular output
  - `executeOnce`: false
- **Key expressions or variables used:**
  - `$json.properties.firstname.value`
  - `$('Code in JavaScript').item.json.text`
- **Input and output connections:**
  - Input: from `Get Contacts`
  - Output: to `Merge` input 2
- **Version-specific requirements:** `typeVersion: 2.3`
- **Edge cases or potential failure types:**
  - Contact schema may differ; some HubSpot outputs use `properties.firstname` instead of `properties.firstname.value`
  - If `firstname.value` is undefined, the filter can fail or produce no matches
  - Contains matching on first names can create multiple ambiguous matches
- **Sub-workflow reference:** none

---

## Block 5 — Merge, AI Generation, and Slack Reply

### Overview
This block consolidates the filtered CRM outputs, sends them to an AI agent powered by Google Gemini, and posts the final generated text to Slack.

### Nodes Involved
- Merge
- Google Gemini Chat Model
- AI Agent
- Send a message

### Node Details

#### 9. Merge
- **Type and technical role:** `n8n-nodes-base.merge`  
  Combines the three filtered branches into a single merged output.
- **Configuration choices:**
  - Mode: `combine`
  - Combine by: `combineByPosition`
  - Number of inputs: 3
  - Include unpaired items: enabled
- **Key expressions or variables used:** none
- **Input and output connections:**
  - Inputs:
    - from `Filter Deals` at input 0
    - from `Filter Companies` at input 1
    - from `Filter Contacts` at input 2
  - Output: to `AI Agent`
- **Version-specific requirements:** `typeVersion: 3.2`
- **Edge cases or potential failure types:**
  - Combining by position may produce unrelated records side by side if each branch returns different counts
  - Include unpaired items can result in sparse merged objects
  - AI prompt quality becomes important because merged structure may be inconsistent
- **Sub-workflow reference:** none

#### 10. Google Gemini Chat Model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatGoogleGemini`  
  Provides the language model used by the AI Agent.
- **Configuration choices:**
  - Temperature: `0.1` for more deterministic output
- **Key expressions or variables used:** none
- **Input and output connections:**
  - Connected to `AI Agent` via `ai_languageModel`
- **Version-specific requirements:** `typeVersion: 1`
- **Edge cases or potential failure types:**
  - Missing or invalid Gemini credentials
  - Model access restrictions
  - API quota exhaustion
  - Regional availability issues depending on Google AI account setup
- **Sub-workflow reference:** none

#### 11. AI Agent
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`  
  Consumes merged CRM data and uses the Gemini model to generate human-readable output.
- **Configuration choices:**
  - No explicit system prompt or custom options shown
  - Receives its language model from `Google Gemini Chat Model`
- **Key expressions or variables used:**  
  No visible custom expressions in the exported node.
- **Input and output connections:**
  - Main input: from `Merge`
  - AI language model input: from `Google Gemini Chat Model`
  - Output: to `Send a message`
- **Version-specific requirements:** `typeVersion: 3`
- **Edge cases or potential failure types:**
  - With no explicit prompt, response formatting may be inconsistent
  - Agent behavior depends on how incoming merged JSON is interpreted
  - If merged data is empty, the agent may hallucinate unless instructed otherwise
  - Token limits may be reached if HubSpot returns many records
- **Sub-workflow reference:** none

#### 12. Send a message
- **Type and technical role:** `n8n-nodes-base.slack`  
  Sends the AI-generated response to a Slack channel.
- **Configuration choices:**
  - Sends text from `{{ $json.output }}`
  - Select mode: `channel`
  - Channel ID fixed to `YOUR_SLACK_CHANNEL_ID`
- **Key expressions or variables used:**
  - `$json.output`
- **Input and output connections:**
  - Input: from `AI Agent`
  - Output: none
- **Version-specific requirements:** `typeVersion: 2.4`
- **Edge cases or potential failure types:**
  - `output` may be empty if AI Agent does not return that field
  - Bot may not have permission to post in the selected channel
  - Hardcoded channel ID means responses are not automatically sent back to the originating conversation unless customized
- **Sub-workflow reference:** none

---

## Block 6 — Documentation and In-Canvas Notes

### Overview
These nodes are visual documentation only. They do not participate in execution, but they explain the workflow purpose, setup, and logical grouping.

### Nodes Involved
- Sticky Note
- Sticky Note1
- Sticky Note2
- Sticky Note3

### Node Details

#### 13. Sticky Note
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  General workflow documentation displayed on the canvas.
- **Configuration choices:**  
  Large note containing workflow description, setup steps, and customization tips.
- **Input and output connections:** none
- **Version-specific requirements:** `typeVersion: 1`
- **Edge cases or potential failure types:** none
- **Sub-workflow reference:** none

#### 14. Sticky Note1
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Labels the trigger and cleanup block.
- **Configuration choices:**  
  Documents Step 1.
- **Input and output connections:** none
- **Version-specific requirements:** `typeVersion: 1`
- **Edge cases or potential failure types:** none
- **Sub-workflow reference:** none

#### 15. Sticky Note2
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Labels the CRM retrieval block.
- **Configuration choices:**  
  Documents Step 2.
- **Input and output connections:** none
- **Version-specific requirements:** `typeVersion: 1`
- **Edge cases or potential failure types:** none
- **Sub-workflow reference:** none

#### 16. Sticky Note3
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Labels the AI response block.
- **Configuration choices:**  
  Documents Step 3.
- **Input and output connections:** none
- **Version-specific requirements:** `typeVersion: 1`
- **Edge cases or potential failure types:** none
- **Sub-workflow reference:** none

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Slack Trigger | n8n-nodes-base.slackTrigger | Entry point; listens for Slack app mentions |  | Code in JavaScript | ## Step 1: Trigger & Clean<br>Captures Slack mentions and removes IDs to extract clean user queries. |
| Code in JavaScript | n8n-nodes-base.code | Removes Slack `<...>` tokens from message text | Slack Trigger | Get many companies, Get Contacts, Get Deals | ## Step 1: Trigger & Clean<br>Captures Slack mentions and removes IDs to extract clean user queries. |
| Get Deals | n8n-nodes-base.hubspot | Retrieves HubSpot deal data | Code in JavaScript | Filter Deals | ## Step 2: CRM Retrieval<br>Fetches and filters HubSpot deals, companies, and contacts. |
| Filter Deals | n8n-nodes-base.filter | Filters deals by deal name against cleaned query | Get Deals | Merge | ## Step 2: CRM Retrieval<br>Fetches and filters HubSpot deals, companies, and contacts. |
| Get many companies | n8n-nodes-base.hubspot | Retrieves all HubSpot companies | Code in JavaScript | Filter Companies | ## Step 2: CRM Retrieval<br>Fetches and filters HubSpot deals, companies, and contacts. |
| Filter Companies | n8n-nodes-base.filter | Filters companies by numeric company ID equality | Get many companies | Merge | ## Step 2: CRM Retrieval<br>Fetches and filters HubSpot deals, companies, and contacts. |
| Get Contacts | n8n-nodes-base.hubspot | Retrieves all HubSpot contacts | Code in JavaScript | Filter Contacts | ## Step 2: CRM Retrieval<br>Fetches and filters HubSpot deals, companies, and contacts. |
| Filter Contacts | n8n-nodes-base.filter | Filters contacts by first name against cleaned query | Get Contacts | Merge | ## Step 2: CRM Retrieval<br>Fetches and filters HubSpot deals, companies, and contacts. |
| Merge | n8n-nodes-base.merge | Combines filtered deals, companies, and contacts | Filter Deals, Filter Companies, Filter Contacts | AI Agent | ## Step 3: AI Response<br>Generates formatted output and sends reply back to Slack. |
| Google Gemini Chat Model | @n8n/n8n-nodes-langchain.lmChatGoogleGemini | Supplies LLM to the AI Agent |  | AI Agent | ## Step 3: AI Response<br>Generates formatted output and sends reply back to Slack. |
| AI Agent | @n8n/n8n-nodes-langchain.agent | Generates natural-language CRM summary | Merge, Google Gemini Chat Model | Send a message | ## Step 3: AI Response<br>Generates formatted output and sends reply back to Slack. |
| Send a message | n8n-nodes-base.slack | Posts final AI response to Slack | AI Agent |  | ## Step 3: AI Response<br>Generates formatted output and sends reply back to Slack. |
| Sticky Note | n8n-nodes-base.stickyNote | Canvas documentation |  |  | ### How it works<br>This workflow creates a Slack-based CRM assistant that lets users query HubSpot data using natural language. When a user mentions the bot in Slack, the workflow captures the message and removes Slack-specific formatting (like mentions and IDs) using a JavaScript node.<br><br>It then retrieves data from HubSpot across Deals, Companies, and Contacts. Each dataset is filtered based on the cleaned query using flexible matching logic. The results are merged and passed to an AI Agent, which formats a clean, structured summary.<br><br>Finally, the formatted response is sent back to Slack as a readable message.<br><br>### Setup steps<br>1. Connect Slack credentials (Bot Token with app_mention and chat:write scopes).<br>2. Invite the bot to your Slack channel using /invite @BotName.<br>3. Connect HubSpot App Token credentials.<br>4. Configure Google Gemini API credentials.<br>5. Test with a Slack mention like: @Bot deal name.<br><br>### Customization tips<br>- Adjust filters to improve matching accuracy.<br>- Modify AI prompt for different output formats.<br>- Add additional HubSpot fields if needed. |
| Sticky Note1 | n8n-nodes-base.stickyNote | Canvas documentation for Step 1 |  |  | ## Step 1: Trigger & Clean<br>Captures Slack mentions and removes IDs to extract clean user queries. |
| Sticky Note2 | n8n-nodes-base.stickyNote | Canvas documentation for Step 2 |  |  | ## Step 2: CRM Retrieval<br>Fetches and filters HubSpot deals, companies, and contacts. |
| Sticky Note3 | n8n-nodes-base.stickyNote | Canvas documentation for Step 3 |  |  | ## Step 3: AI Response<br>Generates formatted output and sends reply back to Slack. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - In n8n, create a blank workflow.
   - Name it: **Build a Slack-based CRM assistant with HubSpot and Google Gemini**.

2. **Create the Slack trigger**
   - Add a **Slack Trigger** node.
   - Set the trigger event to **app_mention**.
   - Enable **Watch Workspace**.
   - Configure Slack credentials using a bot token.
   - Ensure the Slack app has at least:
     - `app_mentions:read`
     - `chat:write`
   - In Slack, install the app to the workspace and invite the bot to the test channel with `/invite @BotName`.

3. **Add the query-cleaning code node**
   - Add a **Code** node after the Slack Trigger.
   - Choose JavaScript mode.
   - Paste this logic:
     ```javascript
     for (const item of $input.all()) {
       const dynamicMentionRegex = /<[^>]+>/g;

       if (item.json.text) {
         item.json.text = item.json.text.replace(dynamicMentionRegex, '').trim();
       }
     }

     return $input.all();
     ```
   - Connect:
     - `Slack Trigger` → `Code in JavaScript`

4. **Add the HubSpot deals node**
   - Add a **HubSpot** node named **Get Deals**.
   - Configure:
     - Resource: **Deal**
     - Operation: **Search**
     - Authentication: **App Token**
   - Connect HubSpot credentials.
   - Connect:
     - `Code in JavaScript` → `Get Deals`

5. **Add the deal filter**
   - Add a **Filter** node named **Filter Deals**.
   - Set condition:
     - Type: **String**
     - Operation: **Contains**
     - Left value: `{{ $json.properties.dealname }}`
     - Right value: `{{ $('Code in JavaScript').item.json.text }}`
   - Enable case-insensitive matching.
   - Set node error behavior to continue regular output if desired.
   - Connect:
     - `Get Deals` → `Filter Deals`

6. **Add the HubSpot companies node**
   - Add another **HubSpot** node named **Get many companies**.
   - Configure:
     - Resource: **Company**
     - Operation: **Get All**
     - Return All: enabled
     - Authentication: **App Token**
   - Connect:
     - `Code in JavaScript` → `Get many companies`

7. **Add the company filter**
   - Add a **Filter** node named **Filter Companies**.
   - Set condition:
     - Type: **Number**
     - Operation: **Equals**
     - Left value: `{{ $json.companyId }}`
     - Right value: `{{ $('Code in JavaScript').item.json.text }}`
   - Disable loose type validation if available.
   - Set error handling to continue regular output.
   - Connect:
     - `Get many companies` → `Filter Companies`
   - Note: this means users must provide a numeric company ID, not a company name.

8. **Add the HubSpot contacts node**
   - Add another **HubSpot** node named **Get Contacts**.
   - Configure:
     - Operation: **Get All**
     - Return All: enabled
     - Authentication: **App Token**
   - Connect:
     - `Code in JavaScript` → `Get Contacts`

9. **Add the contact filter**
   - Add a **Filter** node named **Filter Contacts**.
   - Set condition:
     - Type: **String**
     - Operation: **Contains**
     - Left value: `{{ $json.properties.firstname.value }}`
     - Right value: `{{ $('Code in JavaScript').item.json.text }}`
   - Enable case-insensitive matching.
   - Set error handling to continue regular output.
   - Connect:
     - `Get Contacts` → `Filter Contacts`

10. **Add the merge node**
    - Add a **Merge** node named **Merge**.
    - Configure:
      - Mode: **Combine**
      - Combine by: **Position**
      - Number of inputs: **3**
      - Include unpaired items: enabled
    - Connect:
      - `Filter Deals` → `Merge` input 1
      - `Filter Companies` → `Merge` input 2
      - `Filter Contacts` → `Merge` input 3

11. **Add the Gemini model node**
    - Add a **Google Gemini Chat Model** node.
    - Configure Google Gemini credentials.
    - Set **Temperature** to `0.1`.

12. **Add the AI agent**
    - Add an **AI Agent** node.
    - Connect:
      - `Merge` → `AI Agent` main input
      - `Google Gemini Chat Model` → `AI Agent` language model input
    - Leave default options if reproducing exactly from the exported workflow.
    - Recommended improvement: add a system instruction telling the agent to:
      - summarize only the provided CRM data
      - state clearly when no records match
      - avoid fabricating missing fields

13. **Add the Slack send-message node**
    - Add a **Slack** node named **Send a message**.
    - Configure:
      - Operation to send a message
      - Destination mode: **Channel**
      - Channel ID: your chosen target Slack channel
      - Text: `{{ $json.output }}`
    - Use Slack credentials with `chat:write`.
    - Connect:
      - `AI Agent` → `Send a message`

14. **Add optional sticky notes**
    - Add sticky notes to document the flow:
      - Step 1: Trigger & Clean
      - Step 2: CRM Retrieval
      - Step 3: AI Response
      - General workflow explanation and setup notes

15. **Credential setup checklist**
    - **Slack**
      - Bot token
      - Scopes: `app_mentions:read`, `chat:write`
      - Event subscriptions enabled
      - Bot invited to channel
    - **HubSpot**
      - Private app token or supported app token credential
      - Access to deals, companies, and contacts
    - **Google Gemini**
      - Valid API key or supported credential type in n8n
      - Access to Gemini chat models

16. **Test the workflow**
    - Activate the workflow if using live Slack events.
    - Mention the bot in Slack, for example:
      - `@Bot Acme`
      - `@Bot John`
      - `@Bot 123456`
   - Expectation:
     - Mention is captured
     - Query is cleaned
     - HubSpot branches run in parallel
     - Filtered records are merged
     - AI Agent creates output
     - Slack receives the final message

17. **Important implementation notes for faithful reproduction**
    - The current company matching expects a numeric ID.
    - The current contact filter expects `properties.firstname.value`, which may require adjustment depending on actual HubSpot output.
    - The AI Agent has no visible custom prompt in this export, so output consistency may vary.
    - The Slack reply is sent to a fixed channel, not necessarily the same thread or source channel of the original mention.

18. **Recommended hardening after rebuild**
    - Add a fallback branch when no CRM record matches.
    - Limit HubSpot results instead of retrieving all contacts and companies where possible.
    - Replace company ID equality with company name matching if desired.
    - Add prompt instructions to ensure deterministic formatting.
    - Send the response back to the originating channel or thread using event metadata from Slack.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| This workflow creates a Slack-based CRM assistant that lets users query HubSpot data using natural language. | General workflow purpose |
| When a user mentions the bot in Slack, the workflow captures the message and removes Slack-specific formatting using a JavaScript node. | Processing overview |
| It then retrieves data from HubSpot across Deals, Companies, and Contacts. | CRM integration overview |
| The results are merged and passed to an AI Agent, which formats a clean, structured summary. | AI processing overview |
| Finally, the formatted response is sent back to Slack as a readable message. | Output delivery overview |
| Connect Slack credentials with Bot Token and app_mention/chat:write-related permissions. | Setup guidance |
| Invite the bot to your Slack channel using `/invite @BotName`. | Slack channel setup |
| Connect HubSpot App Token credentials. | Setup guidance |
| Configure Google Gemini API credentials. | Setup guidance |
| Test with a Slack mention like: `@Bot deal name`. | Validation example |
| Adjust filters to improve matching accuracy. | Customization guidance |
| Modify AI prompt for different output formats. | Customization guidance |
| Add additional HubSpot fields if needed. | Customization guidance |

**Disclaimer:** Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.