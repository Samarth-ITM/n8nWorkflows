Xano Support Ticket Router (AI + Xano Node Integration)

https://n8nworkflows.xyz/workflows/xano-support-ticket-router--ai---xano-node-integration--11613


# Xano Support Ticket Router (AI + Xano Node Integration)

### 1. Workflow Overview

This workflow, titled **Xano Support Ticket Router (AI + Xano Node Integration)**, is designed to automate the intake, classification, and routing of support tickets using AI and backend logic managed via Xano. It targets use cases where support requests come from various sources and need to be programmatically classified and stored in a backend database for further processing or escalation.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Receives incoming support ticket data via a webhook.
- **1.2 AI Processing and Classification:** Uses an AI agent powered by OpenAI to analyze and classify the incoming ticket content.
- **1.3 User Existence Check:** Queries the Xano backend to determine if the ticket submitter already exists in the database.
- **1.4 Conditional Branching & Record Creation:** Depending on user existence, either creates a new user and their support ticket or creates just the support ticket.
- **1.5 Ticket Routing and Further Automation:** Returns the processed data for additional automations or integrations as needed.

### 2. Block-by-Block Analysis

---

#### 2.1 Input Reception

- **Overview:**  
  This block captures incoming support ticket requests via a webhook, serving as the entry point for the workflow.

- **Nodes Involved:**  
  - Ticket Ingestion (Webhook)
  - Sticky Note (Web Request Explanation)

- **Node Details:**

  - **Ticket Ingestion**  
    - Type: Webhook  
    - Role: Receives POST requests with raw ticket data payloads.  
    - Configuration:  
      - HTTP Method: POST  
      - Path: fa4b1026-bc47-4d9d-b202-7c97f0d1b9e2 (unique webhook ID)  
    - Inputs: External HTTP POST requests  
    - Outputs: JSON payload representing the incoming ticket data  
    - Edge Cases: Misformatted requests, missing expected fields, or unauthorized sources if webhook security is not configured.

  - **Sticky Note (Web Request Explanation)**  
    - Type: Documentation node  
    - Content: Explains that this webhook simulates or listens to actual ticket platform data.  
    - Role: Provides context for users configuring or testing the webhook.

---

#### 2.2 AI Processing and Classification

- **Overview:**  
  This block processes the raw ticket data using a Langchain-based AI agent leveraging OpenAI, classifying the support request into predefined categories.

- **Nodes Involved:**  
  - OpenAI Model  
  - Agent  
  - Sticky Note (Listening Request Explanation)

- **Node Details:**

  - **OpenAI Model**  
    - Type: Langchain OpenAI Chat Language Model Node  
    - Role: Provides language model capabilities to the Agent node.  
    - Configuration: Default options with no specific parameters altered.  
    - Inputs: Receives prompts from the Agent node.  
    - Outputs: AI-generated responses used by the Agent.

  - **Agent**  
    - Type: Langchain Agent Node  
    - Role: Core AI logic that receives the user message and chat history, classifies the support request into one of six categories via a system message prompt.  
    - Configuration:  
      - Input Text: Extracted from `$json.body` of the incoming payload.  
      - System Message: Defines classification categories and instructs the agent to respond with JSON only.  
      - Prompt Type: "Define" (custom prompt with system instructions)  
    - Key Expressions:  
      - `{{$json.body}}` for user input data  
      - `{{$json.chatInput}}` and `{{$json.chatHistory}}` for conversational context  
    - Inputs: Receives webhook data  
    - Outputs: JSON structured classification result  
    - Edge Cases: Failure to parse JSON, unexpected response format, OpenAI API rate limits or timeouts.

  - **Sticky Note (Listening Request Explanation)**  
    - Type: Documentation node  
    - Content: Explains that once Xano completes its processing, data returns to n8n for further automation steps.

---

#### 2.3 User Existence Check

- **Overview:**  
  Queries Xano backend to determine if the user associated with the support ticket already exists in the database.

- **Nodes Involved:**  
  - Search row (Xano)  
  - Sticky Note (User existence check explanation)

- **Node Details:**

  - **Search row**  
    - Type: Xano Node (Preview)  
    - Role: Searches user records in the Xano backend to check for existing users based on criteria from the ticket data.  
    - Configuration: Uses Xano credentials and configured to perform a search query on the user table.  
    - Inputs: Output from Agent node (classification and extracted data)  
    - Outputs: Result set indicating user existence (number of items found)  
    - Edge Cases: Authentication errors with Xano, empty or malformed search criteria, network timeouts.

  - **Sticky Note (User existence check explanation)**  
    - Content: Explains the logic of checking if the user exists to decide whether to create a new user or proceed directly to ticket creation.

---

#### 2.4 Conditional Branching & Record Creation

- **Overview:**  
  Branches execution based on whether the user exists. If no user is found, creates a new user and then a support ticket; otherwise, creates a support ticket directly.

- **Nodes Involved:**  
  - If (Condition)  
  - Create User (Xano)  
  - _Create Support Ticket (Xano)  
  - Create Support Ticket_ (Xano)  
  - Sticky Notes (General setup and branch explanation)

- **Node Details:**

  - **If**  
    - Type: Conditional (If) Node  
    - Role: Checks if the number of items found in the Search row node is zero (user does not exist).  
    - Configuration:  
      - Condition: `$json.itemsReceived === 0`  
      - Combinator: AND (only one condition)  
    - Inputs: Output from Search row  
    - Outputs: Two branches  
      - True: User not found  
      - False: User found  
    - Edge Cases: Missing `itemsReceived` field, type mismatches.

  - **Create User**  
    - Type: Xano Node (Preview)  
    - Role: Creates a new user record in Xano.  
    - Configuration: Uses Xano credentials, target table, and create operation.  
    - Inputs: True branch from If node  
    - Outputs: Newly created user data passed to subsequent ticket creation  
    - Edge Cases: Authentication failure, validation errors, duplicate user creation.

  - **_Create Support Ticket**  
    - Type: Xano Node (Preview)  
    - Role: Creates a support ticket in Xano tied to the newly created user.  
    - Configuration: Uses Xano credentials, target table, and create operation.  
    - Inputs: Output from Create User node  
    - Outputs: Confirmation or record data  
    - Edge Cases: Backend validation failures, connectivity issues.

  - **Create Support Ticket_**  
    - Type: Xano Node (Preview)  
    - Role: Creates a support ticket directly for existing users.  
    - Configuration: Uses Xano credentials, target table, and create operation.  
    - Inputs: False branch from If node  
    - Outputs: Confirmation or record data  
    - Edge Cases: Similar to _Create Support Ticket node.

  - **Sticky Notes:**  
    - Provide detailed instructions on how to set up and connect the Xano nodes, credentials, and the conditional logic for user and ticket creation.

---

#### 2.5 Ticket Routing and Further Automation

- **Overview:**  
  Although not explicitly detailed in additional nodes, the workflow is designed to return enriched and structured ticket data for further automations such as notifications, CRM updates, or escalation workflows.

- **Nodes Involved:**  
  - Agent Response (Webhook)  
  - Sticky Note (General workflow explanation and setup)

- **Node Details:**

  - **Agent Response**  
    - Type: Webhook  
    - Role: Acts as an endpoint to receive or send processed data back for external systems or services.  
    - Configuration:  
      - HTTP Method: POST  
      - Path: dfc07bd5-0904-4deb-b124-926861a65abe  
    - Inputs/Outputs: Can be used to receive or send processed ticket data  
    - Edge Cases: Missing authentication, data format mismatches.

  - **Sticky Note (General workflow explanation and setup)**  
    - Content:  
      - Comprehensive instructions on how the workflow functions step-by-step  
      - How to configure Xano credentials and nodes  
      - Links to Xano snippets and setup guides  
    - Role: Serves as a detailed reference and onboarding guide for developers.

---

### 3. Summary Table

| Node Name           | Node Type                         | Functional Role                            | Input Node(s)          | Output Node(s)            | Sticky Note                                                                                                                       |
|---------------------|----------------------------------|-------------------------------------------|-----------------------|---------------------------|----------------------------------------------------------------------------------------------------------------------------------|
| Ticket Ingestion     | Webhook                          | Receive incoming support ticket data      | External HTTP POST    | Agent                     | ## Webhook request - Simulate a webhook request through your Xano Function, or configure to listen to actual data from your ticketing platform of choice. |
| Agent               | Langchain Agent                  | Classify and analyze support request      | Ticket Ingestion      | Search row                | ## Listening request - Once Xano is done performing the analysis, the returned data is brought back into n8n for further automations! |
| OpenAI Model         | Langchain OpenAI Chat Model      | Provides AI model capabilities for Agent  | Agent (ai_languageModel) | Agent                     |                                                                                                                                  |
| Search row           | Xano Node (Preview)              | Search for existing user in Xano backend | Agent                 | If                        | ## Check if a user exists, then do something! - If this request is coming from a new user, create user then ticket. Else create ticket only. |
| If                   | Conditional (If)                 | Branch logic based on user existence       | Search row            | Create User, Create Support Ticket_ |                                                                                                                                  |
| Create User          | Xano Node (Preview)              | Create a new user record                   | If (true)             | _Create Support Ticket     |                                                                                                                                  |
| _Create Support Ticket| Xano Node (Preview)              | Create ticket for newly created user       | Create User            |                           |                                                                                                                                  |
| Create Support Ticket_| Xano Node (Preview)              | Create ticket for existing user             | If (false)            |                           |                                                                                                                                  |
| Agent Response       | Webhook                         | Endpoint for sending/receiving processed data |                        |                           |                                                                                                                                  |
| Sticky Note          | Sticky Note                     | Documentation and instructions             |                       |                           | # How It Works - Full workflow explanation and setup guide including credential configuration and Xano integration instructions. |
| Sticky Note1         | Sticky Note                     | Contains external video link                |                       |                           | @[youtube](XLXCS-USO3Y)                                                                                                         |
| Sticky Note3         | Sticky Note                     | Explanation of data listening post-Xano processing |                       |                           | ## Listening request - Once Xano is done performing the analysis, the returned data is brought back into n8n for further automations! |
| Sticky Note4         | Sticky Note                     | Detailed overview and setup instructions    |                       |                           | # How It Works and How to Set It Up including link to Xano snippet: https://www.xano.com/snippet/sZlboSHV                      |
| Sticky Note6         | Sticky Note                     | Explains user existence check logic         |                       |                           | ## Check if a user exists, then do something!                                                                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node “Ticket Ingestion”**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `fa4b1026-bc47-4d9d-b202-7c97f0d1b9e2` (or custom)  
   - Purpose: Receive incoming support ticket data.

2. **Create Langchain OpenAI Model Node “OpenAI Model”**  
   - Type: `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
   - Configuration: Leave default options (no special parameters).  
   - Purpose: Provide OpenAI chat capabilities.

3. **Create Langchain Agent Node “Agent”**  
   - Type: `@n8n/n8n-nodes-langchain.agent`  
   - Parameters:  
     - Text input: `={{ $json.body }}` (extract ticket message)  
     - System Message:  
       ```
       You are a support ticket intake and triage assistant. Your job is to collect user information and classify support requests.

       CLASSIFICATION CATEGORIES:
       - billing: Payment issues, invoices, subscriptions, refunds, pricing
       - product_question: How-to questions, feature usage, general inquiries
       - bug_report: Errors, broken features, technical malfunctions
       - cancellation_risk: Intent to cancel, threats to leave, competitor mentions
       - feature_request: Requests for new features or enhancements
       - other: Anything else

       CONVERSATION CONTEXT:
       User Message: {{ $json.chatInput }}
       Chat History: {{ $json.chatHistory || 'None' }}

       NOW ANALYZE THE CURRENT CONVERSATION AND RESPOND WITH JSON ONLY:
       ```
     - Prompt Type: Define  
   - Connect “Ticket Ingestion” main output to “Agent” main input.  
   - Connect “OpenAI Model” to “Agent” ai_languageModel input.

4. **Create Xano Node “Search row”**  
   - Type: `@xano/n8n-nodes-preview-xano.xano`  
   - Operation: Search Rows in User table  
   - Configure with Xano credentials (base URL + access token).  
   - Input: Connect from “Agent” main output.  
   - Purpose: Check if user exists.

5. **Create Conditional Node “If”**  
   - Type: If  
   - Condition: Check if `{{$json.itemsReceived}} === 0`  
   - Input: Connect from “Search row” main output.  
   - Outputs: two branches (True = No user, False = User exists).

6. **Create Xano Node “Create User”**  
   - Type: Xano Node  
   - Operation: Create row in User table  
   - Input: Connect True output of “If” node.  
   - Configure fields with user data from previous nodes.

7. **Create Xano Node “_Create Support Ticket”**  
   - Type: Xano Node  
   - Operation: Create row in Support Ticket table  
   - Input: Connect from “Create User” node.  
   - Purpose: Create ticket for newly created user.

8. **Create Xano Node “Create Support Ticket_”**  
   - Type: Xano Node  
   - Operation: Create row in Support Ticket table  
   - Input: Connect False output of “If” node.  
   - Purpose: Create ticket for existing user.

9. **Create Webhook Node “Agent Response”** (Optional)  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `dfc07bd5-0904-4deb-b124-926861a65abe` (or custom)  
   - Purpose: Endpoint for returning or receiving processed ticket data.

10. **Add Sticky Notes** for documentation within n8n editor, including:  
    - Explanation of webhook setup  
    - AI processing description  
    - User existence logic  
    - General workflow overview and setup instructions  
    - Link to [Xano snippet](https://www.xano.com/snippet/sZlboSHV) for backend setup  
    - YouTube video reference: (XLXCS-USO3Y)

---

### 5. General Notes & Resources

| Note Content                                                                                                       | Context or Link                                            |
|--------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------|
| Comprehensive workflow setup and execution instructions covering data reception, AI classification, Xano integration, and automation continuation. | Embedded in Sticky Note “Sticky Note4” in the workflow.    |
| Xano snippet for backend logic to use with this workflow: https://www.xano.com/snippet/sZlboSHV                     | Official Xano snippet referenced in workflow documentation. |
| YouTube video for additional guidance: @[youtube](XLXCS-USO3Y)                                                    | Linked in Sticky Note1 node for visual walkthrough.        |
| This workflow requires valid Xano credentials (base URL and access token) configured within n8n’s credential manager.| Essential for connecting to Xano backend nodes.             |
| AI classification depends on OpenAI API access; ensure API key and rate limits are handled appropriately.          | Required for Agent and OpenAI model nodes.                  |

---

**Disclaimer:**  
The provided workflow content strictly adheres to content policies and only processes legal and public data. It contains no illegal or protected information and is intended solely for lawful automation and integration purposes.