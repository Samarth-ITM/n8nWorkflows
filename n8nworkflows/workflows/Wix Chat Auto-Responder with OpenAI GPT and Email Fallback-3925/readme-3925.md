Wix Chat Auto-Responder with OpenAI GPT and Email Fallback

https://n8nworkflows.xyz/workflows/wix-chat-auto-responder-with-openai-gpt-and-email-fallback-3925


# Wix Chat Auto-Responder with OpenAI GPT and Email Fallback

### 1. Workflow Overview

This workflow automates responses to Wix Chat messages using OpenAI GPT models, with a fallback to email alerts for human intervention. It is designed to intelligently handle incoming chat messages by distinguishing between members and anonymous visitors, checking recent human responses to avoid redundant AI replies, and splitting long AI-generated messages to fit Wix API limits. The workflow is modular and extensible, suitable for solopreneurs, agencies, or support teams looking to automate chat while maintaining control and fallback paths.

**Logical Blocks:**

- **1.1 Input Reception & Authentication:** Receiving chat messages via a webhook, acquiring OAuth tokens for Wix API access.
- **1.2 Visitor Type Check & Throttle Logic:** Determining if the user is a member or visitor and whether AI should respond based on recent human interaction.
- **1.3 Chat History Retrieval:** Fetching conversation history from Wix to provide context for AI replies.
- **1.4 AI Response Generation:** Using GPT-4 models to generate chat replies with context memory buffers.
- **1.5 Message Chunking & Sending:** Splitting AI messages and sending them back through Wix Chat API.
- **1.6 Email Fallback Alert:** Sending emails to support if AI should not answer or human intervention is needed.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Authentication

- **Overview:** This block receives incoming chat messages from Wix via webhook and retrieves OAuth tokens needed for authenticated API calls.
- **Nodes Involved:** Webhook, Execute Workflow, HTTP Request1, HTTP Request2, HTTP Request3, HTTP Request4, HTTP Request5, Code1
- **Node Details:**

  - **Webhook**
    - Type: Webhook (Trigger)
    - Configuration: Listens for Wix Chat messages via a publicly exposed URL.
    - Input: Incoming Wix chat webhook payload.
    - Output: Passes payload to Execute Workflow node.
    - Failures: Webhook misconfiguration, network issues.

  - **Execute Workflow**
    - Type: Execute Workflow
    - Role: Runs a sub-workflow to generate OAuth tokens for Wix API.
    - Input: Incoming webhook data.
    - Output: OAuth tokens for subsequent API calls.
    - Requirements: Sub-workflow must be configured with Wix credentials.
    - Failures: Credential errors, sub-workflow failures.

  - **HTTP Request1**
    - Type: HTTP Request
    - Role: Makes authenticated API calls to Wix (e.g., verify or fetch data).
    - Configuration: Uses OAuth token.
    - Input: Token and webhook data.
    - Output: API response forwarded for logic decisions.
    - Failures: Auth errors, 4xx/5xx HTTP errors, token expiration.

  - **HTTP Request2, HTTP Request3, HTTP Request4, HTTP Request5**
    - Type: HTTP Request
    - Role: Various API calls to Wix for fetching chat history or sending messages.
    - Configuration: Uses OAuth tokens, Wix API endpoints.
    - Input/Output: Chained for data retrieval and processing.
    - Failures: Network, auth, or API schema changes.

  - **Code1**
    - Type: Code (JavaScript)
    - Role: Processes API response data, often for conditional checks.
    - Input: Responses from HTTP nodes.
    - Output: Boolean or filtered data for If nodes.
    - Failures: Expression errors or unexpected data structure.

#### 1.2 Visitor Type Check & Throttle Logic

- **Overview:** Determines if the chat user is a registered member or anonymous visitor and checks if a human has responded recently to avoid redundant AI replies.
- **Nodes Involved:** If (renamed Check Member vs Visitor), Code, If2, Human response within last hour, END1, End chat
- **Node Details:**

  - **If (Check Member vs Visitor)**
    - Type: If
    - Role: Branches workflow based on visitor type extracted from webhook payload.
    - Input: Webhook data.
    - Output: Routes to member or visitor-specific processing.
    - Failures: Expression errors or missing payload fields.

  - **Code (Response Throttle)**
    - Type: Code
    - Role: Implements throttle logic to allow AI response only if no human reply occurred in last 12 hours.
    - Input: Chat history timestamps.
    - Output: Boolean flags for AI response or fallback.
    - Failures: Timing calculation errors.

  - **If2**
    - Type: If
    - Role: Checks if a human response exists in the recent timeframe.
    - Output: Ends chat if human response found, else proceeds to AI response generation.
    - Failures: Logical errors in condition.

  - **Human response within last hour, END1 & End chat**
    - Type: NoOp
    - Role: Ends the workflow early if conditions indicate AI should not respond.
    - Input: Branch from If nodes.
    - Output: Terminates this execution cleanly.

#### 1.3 Chat History Retrieval

- **Overview:** Fetches recent chat messages from Wix to provide AI with conversation context.
- **Nodes Involved:** HTTP Request2, HTTP Request3, HTTP Request4, HTTP Request5, Code1
- **Node Details:**

  - **HTTP Request2 - HTTP Request5**
    - Role: Sequentially fetch paginated chat history or relevant conversation details.
    - Configuration: Uses OAuth token, Wix chat API endpoints.
    - Output: Chat history data.
    - Failures: Pagination issues, API limits.

  - **Code1**
    - Role: Processes combined chat history into a format suitable for AI context.
    - Output: Structured messages for AI prompt.
    - Failures: Data parsing errors.

#### 1.4 AI Response Generation

- **Overview:** Generates responses using OpenAI GPT-4 models with memory buffers to keep context.
- **Nodes Involved:** AI Agent, AI Agent1, OpenAI Chat Model1, OpenAI Chat Model2, Window Buffer Memory, Window Buffer Memory1, Code, Code5, Code6, If1
- **Node Details:**

  - **AI Agent & AI Agent1**
    - Type: LangChain Agent nodes
    - Role: Orchestrate prompt construction, memory management, and model calling.
    - Input: Processed chat history and user input.
    - Output: Generated AI text.
    - Requirements: Configured with OpenAI API keys, model type (GPT-4/GPT-4o).
    - Failures: API rate limits, invalid prompts.

  - **OpenAI Chat Model1 & OpenAI Chat Model2**
    - Type: LangChain OpenAI Chat Model
    - Role: Connect to OpenAI GPT-4 or GPT-4o models.
    - Configuration: Model version, temperature, API key.
    - Output: AI-generated reply chunks.
    - Failures: API key invalid, network issues.

  - **Window Buffer Memory & Window Buffer Memory1**
    - Type: LangChain Memory Buffer
    - Role: Maintain conversation context window for AI.
    - Input/Output: Message history for agents.
    - Failures: Memory size constraints.

  - **Code, Code5, Code6**
    - Type: Code nodes
    - Role: Post-process AI output, chunk splitting, or control flow.
    - Failures: Logic errors in chunking or formatting.

  - **If1**
    - Type: If
    - Role: Decide if AI response should be sent or the chat ended.
    - Output: Routes to AI reply sending or workflow termination.

#### 1.5 Message Chunking & Sending

- **Overview:** Splits long AI messages into smaller chunks and sends them back to Wix Chat using API calls.
- **Nodes Involved:** Loop Over Items, Loop Over Items1, HTTP Request6, HTTP Request
- **Node Details:**

  - **Loop Over Items & Loop Over Items1**
    - Type: SplitInBatches
    - Role: Iterate over message chunks to send them sequentially.
    - Input: AI-generated message chunks.
    - Output: Single message chunk per iteration.
    - Failures: Loop break conditions, empty data.

  - **HTTP Request6 & HTTP Request**
    - Type: HTTP Request
    - Role: Send each message chunk to Wix Chat API.
    - Configuration: OAuth token, API endpoint for sending messages.
    - Failures: API limits, auth errors, message size constraints.

#### 1.6 Email Fallback Alert

- **Overview:** Sends an email notification to support staff if AI is not allowed to respond or human intervention is required.
- **Nodes Involved:** Send Email, Send Email1
- **Node Details:**

  - **Send Email & Send Email1**
    - Type: Email Send Tool
    - Role: Dispatch fallback alert emails.
    - Configuration: SMTP credentials, recipient addresses, customizable message.
    - Input: Triggered from AI Agent nodes when fallback is necessary.
    - Failures: SMTP auth errors, invalid email addresses.

---

### 3. Summary Table

| Node Name                      | Node Type                       | Functional Role                                   | Input Node(s)                 | Output Node(s)                  | Sticky Note                                                                                                   |
|--------------------------------|--------------------------------|--------------------------------------------------|------------------------------|--------------------------------|---------------------------------------------------------------------------------------------------------------|
| Webhook                        | Webhook                        | Receives Wix Chat messages                        | -                            | Execute Workflow                | Copy this webhook URL to Wix Automations for chat trigger                                                    |
| Execute Workflow               | Execute Workflow               | Generates OAuth token for Wix API                 | Webhook                      | If                             | Can be modularized and called separately                                                                     |
| HTTP Request1                 | HTTP Request                  | Wix API call for verification/data                | When Executed by Another Workflow | Loop Over Items1            | Replace placeholders with your Wix App credentials                                                           |
| HTTP Request2                 | HTTP Request                  | Fetches chat history or context                    | If                           | HTTP Request5                  |                                                                                                               |
| HTTP Request3                 | HTTP Request                  | Additional Wix API call for data                   | If                           | HTTP Request4                  |                                                                                                               |
| HTTP Request4                 | HTTP Request                  | Further data retrieval                             | HTTP Request3                | Code                          |                                                                                                               |
| HTTP Request5                 | HTTP Request                  | Final chat history fetch                           | HTTP Request2                | Code1                         |                                                                                                               |
| Code1                         | Code                          | Processes chat history data                        | HTTP Request5                | If2                           |                                                                                                               |
| If                            | If                            | Checks if visitor is member vs visitor            | Execute Workflow             | HTTP Request2 / HTTP Request3  | Renamed "Check Member vs Visitor"                                                                             |
| Code                         | Code                          | Throttle logic for AI response timing             | HTTP Request4                | If1                           | Modify throttle parameters here                                                                                |
| If1                           | If                            | Decides if AI should respond or end chat          | Code                        | End chat / AI Agent1           |                                                                                                               |
| AI Agent1                    | LangChain Agent               | Generates AI response for one branch              | If1                         | Code5                         | Uses OpenAI Chat Model1 and Window Buffer Memory1                                                             |
| OpenAI Chat Model1            | LangChain OpenAI Chat Model  | Connects to OpenAI GPT-4 model                     | AI Agent1                   | AI Agent1                     | Add your OpenAI API key here                                                                                   |
| Window Buffer Memory1         | LangChain Memory Buffer       | Maintains conversation context for AI Agent1      | OpenAI Chat Model1           | AI Agent1                     |                                                                                                               |
| Code5                        | Code                          | Processes AI output, prepares message chunks       | AI Agent1                   | Loop Over Items                |                                                                                                               |
| Loop Over Items              | SplitInBatches                | Iterates over AI message chunks                    | Code5                       | HTTP Request6 / (empty batch)  |                                                                                                               |
| HTTP Request6                | HTTP Request                  | Sends message chunk to Wix Chat API                | Loop Over Items             | Loop Over Items                |                                                                                                               |
| If2                          | If                            | Checks human response within timeframe             | Code1                       | Human response within last hour, END1 / AI Agent |                                                                                                               |
| Human response within last hour, END1 | NoOp                    | Ends workflow if human responded recently           | If2                         | -                            |                                                                                                               |
| AI Agent                     | LangChain Agent               | Generates AI response for second branch             | If2                         | Code6                         | Uses OpenAI Chat Model2 and Window Buffer Memory                                                              |
| OpenAI Chat Model2            | LangChain OpenAI Chat Model  | Connects to OpenAI GPT-4 (alternate)                | AI Agent                    | AI Agent                     | Add your OpenAI API key here                                                                                   |
| Window Buffer Memory          | LangChain Memory Buffer       | Maintains conversation context for AI Agent         | OpenAI Chat Model2           | AI Agent                     |                                                                                                               |
| Code6                        | Code                          | Processes AI output, prepares message chunks        | AI Agent                    | Loop Over Items1               |                                                                                                               |
| Loop Over Items1             | SplitInBatches                | Iterates over AI message chunks                      | Code6                       | HTTP Request / (empty batch)   |                                                                                                               |
| HTTP Request                 | HTTP Request                  | Sends message chunk to Wix Chat API                  | Loop Over Items1            | Loop Over Items1               |                                                                                                               |
| Send Email                   | Email Send Tool               | Sends fallback email alert                          | AI Agent                   | -                            | Configure SMTP credentials and recipients                                                                      |
| Send Email1                  | Email Send Tool               | Sends fallback email alert (second branch)          | AI Agent1                  | -                            | Same as above                                                                                                   |
| End chat                    | NoOp                          | Terminates workflow when no AI response needed      | If1                         | -                            |                                                                                                               |
| When Executed by Another Workflow | Execute Workflow Trigger   | Sub-workflow trigger for OAuth token generation      | -                            | HTTP Request1                 | Used for modular OAuth token refresh                                                                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: Webhook  
   - Configure to receive POST requests from Wix Chat webhook trigger.  
   - Save the webhook URL for Wix automation setup.

2. **Add Execute Workflow Node**  
   - Type: Execute Workflow  
   - Link it from Webhook node output.  
   - Create or select a sub-workflow to generate OAuth tokens for Wix API with your Wix app credentials (Client ID, Secret, Instance ID).  
   - Ensure sub-workflow outputs the OAuth access token.

3. **Add If Node (Check Member vs Visitor)**  
   - Connect from Execute Workflow output.  
   - Configure condition to check if user is a member or anonymous visitor based on webhook payload fields.

4. **Add HTTP Request Nodes for Chat History**  
   - Add HTTP Request2, HTTP Request3, HTTP Request4, HTTP Request5 sequentially connected.  
   - Configure each to call Wix API endpoints to fetch chat conversation history, passing OAuth token for authentication.

5. **Add Code Node (Code1)**  
   - Connect from HTTP Request5 output.  
   - Implement JavaScript to process chat history messages to format for AI context.

6. **Add If Node (If2)**  
   - Connect from Code1 output.  
   - Configure to check if a human responded within the last 12 hours (configurable).  
   - If yes, connect to NoOp node “Human response within last hour, END1” to end workflow.  
   - If no, proceed to AI Agent node.

7. **Add AI Agent Node**  
   - Connect from If2 no branch.  
   - Configure with OpenAI Chat Model2 and Window Buffer Memory nodes for context memory.  
   - Set OpenAI API key and model (GPT-4 or GPT-4o).  
   - This node generates AI chat response.

8. **Add Code Node (Code6)**  
   - Connect from AI Agent output.  
   - Implement logic to split long AI messages into chunks compliant with Wix API limits.

9. **Add SplitInBatches Node (Loop Over Items1)**  
   - Connect from Code6 output.  
   - Configure batch size = 1 to send message chunks sequentially.

10. **Add HTTP Request Node (HTTP Request)**  
    - Connect from Loop Over Items1 output.  
    - Configure to send each message chunk back to Wix Chat API using OAuth token.

11. **Add Send Email Node (Fallback alert)**  
    - Connect as an AI tool output from AI Agent node.  
    - Configure SMTP credentials, recipient email, and customize alert message for fallback notifications.

12. **Repeat steps 7-11 for alternate branch**  
    - For member vs visitor differentiation, use AI Agent1, OpenAI Chat Model1, Window Buffer Memory1, Code5, Loop Over Items, HTTP Request6, Send Email1 nodes accordingly.

13. **Add If Node (If1) and End chat Node**  
    - From intermediate Code node output, check if AI should respond or end chat.  
    - Connect to AI Agent1 or NoOp “End chat” node accordingly.

14. **Configure all HTTP Request nodes**  
    - Input OAuth token in headers.  
    - Use Wix API endpoints for chat history and message posting.  
    - Handle pagination and error checking.

15. **Set Up Wix Automation**  
    - In Wix dashboard, create new automation triggered by chat message send event.  
    - Add action to send webhook to the n8n Webhook URL.

16. **Test the entire flow**  
    - Use incognito browser on Wix site.  
    - Send chat messages as member and visitor.  
    - Verify AI responds only when no recent human reply exists and fallback emails send properly.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                 | Context or Link                                                                                                            |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------|
| You can modularize OAuth token generation into a separate workflow and call it via Execute Workflow node for cleaner design.                                | Setup Instructions Step 3                                                                                                   |
| Paste the Webhook URL from the Webhook node into Wix Automations under Settings > Automations > Create New > Trigger: "When someone sends a message via chat" | Setup Instructions Step 4                                                                                                   |
| Adjust throttle logic parameters in the Code node labeled "Response Throttle" to customize AI response timing and fallback behavior.                       | Setup Instructions Step 6                                                                                                   |
| You may replace email fallback with Slack, SMS, or webhook alerts by swapping the Send Email node accordingly.                                              | Setup Instructions Step 7                                                                                                   |
| Ensure SMTP credentials are valid and tested before activating fallback email alerts.                                                                        | Requirements                                                                                                               |
| OpenAI API keys must be inserted into both OpenAI Chat Model nodes; models can be GPT-4 or GPT-4o, and temperature can be adjusted to tune responses.         | Setup Instructions Step 5                                                                                                   |
| Sticky notes in the workflow provide additional usage tips and reminders for node renaming and configuration.                                               | Workflow includes sticky notes for clarity                                                                                  |

---

This comprehensive reference enables understanding, modification, and reconstruction of the Wix Chat Auto-Responder workflow integrating OpenAI GPT and email fallback in n8n.