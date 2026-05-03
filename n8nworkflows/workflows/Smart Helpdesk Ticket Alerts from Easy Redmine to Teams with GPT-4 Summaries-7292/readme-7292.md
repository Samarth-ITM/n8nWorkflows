Smart Helpdesk Ticket Alerts from Easy Redmine to Teams with GPT-4 Summaries

https://n8nworkflows.xyz/workflows/smart-helpdesk-ticket-alerts-from-easy-redmine-to-teams-with-gpt-4-summaries-7292


# Smart Helpdesk Ticket Alerts from Easy Redmine to Teams with GPT-4 Summaries

### 1. Workflow Overview

This workflow automates the process of notifying a Microsoft Teams channel whenever a new helpdesk ticket is created in Easy Redmine. It retrieves detailed ticket information, uses an AI language model (GPT-4) to generate a structured summary and suggested solution based on the ticket description, and then sends a formatted message to the designated Teams channel.

The workflow is logically grouped into the following blocks:

- **1.1 Input Reception:** Receives new ticket creation events via a webhook from Easy Redmine.
- **1.2 Ticket Data Retrieval:** Uses the Easy Redmine API to fetch detailed ticket information by ticket ID.
- **1.3 Data Preparation:** Extracts and formats relevant ticket fields, including a direct URL to the ticket.
- **1.4 AI Processing:** Employs GPT-4 via the LangChain agent to summarize the ticket description and suggest a technical solution.
- **1.5 Output Delivery:** Sends the AI-generated summary and ticket details as an HTML message to a Microsoft Teams channel.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block listens for new ticket creation events from Easy Redmine via a webhook and triggers the workflow on each event.

- **Nodes Involved:**  
  - Catch Easy Webhook - New Issue Created

- **Node Details:**

  - **Catch Easy Webhook - New Issue Created**  
    - **Type:** Webhook (HTTP POST listener)  
    - **Role:** Entry point for new ticket creation events from Easy Redmine.  
    - **Configuration:**  
      - HTTP Method: POST  
      - Path: `cc5d8044-9921-4b50-8ad4-9a2f1b6718ec` (unique webhook endpoint)  
    - **Input/Output:**  
      - Input: Incoming POST request from Easy Redmine webhook  
      - Output: JSON containing the payload with new issue data, including `issue.id`  
    - **Version Requirements:** n8n v0.154+ recommended for webhook stability  
    - **Edge Cases:**  
      - Missing or malformed payload could cause failures downstream  
      - Unauthorized requests if webhook URL is exposed or not secured  
    - **Sticky Note Context:** Describes use of Easy Redmine webhook to trigger ticket processing.  

#### 2.2 Ticket Data Retrieval

- **Overview:**  
  Retrieves detailed information about the newly created ticket from Easy Redmine via API using the ticket ID obtained from the webhook.

- **Nodes Involved:**  
  - Get a new ticket by ID

- **Node Details:**

  - **Get a new ticket by ID**  
    - **Type:** Easy Redmine node (API integration)  
    - **Role:** Fetches complete ticket details by ID from Easy Redmine.  
    - **Configuration:**  
      - Operation: Get One Issue  
      - ID: Dynamically set to `{{$json.body.issue.id}}` (from webhook data)  
      - Credentials: Easy Redmine API credentials with appropriate permissions  
    - **Input/Output:**  
      - Input: Ticket ID from webhook  
      - Output: Detailed JSON object with ticket fields (subject, priority, description, etc.)  
    - **Version Requirements:** Compatible with Easy Redmine API v2+  
    - **Edge Cases:**  
      - API call failures due to invalid ID, network issues, or permission errors  
      - Rate limiting by Easy Redmine API  
    - **Sticky Note Context:** Notes importance of using API with technical user credentials.  

#### 2.3 Data Preparation

- **Overview:**  
  Selects relevant fields (description and creates a direct URL) for AI summarization and message formatting.

- **Nodes Involved:**  
  - Pick Description & Create URL to issue

- **Node Details:**

  - **Pick Description & Create URL to issue**  
    - **Type:** Set node (data manipulation)  
    - **Role:** Extracts the `description` from ticket data and constructs a direct URL link to the issue in Easy Redmine.  
    - **Configuration:**  
      - Assignments:  
        - `description` = `{{$json.issue.description}}`  
        - `url link to issue` = `"https://easyredmine-application.com/issues/{{$json.issue.id}}"`  
    - **Input/Output:**  
      - Input: JSON from Easy Redmine node  
      - Output: JSON with simplified fields for AI input and message generation  
    - **Version Requirements:** n8n v0.142+ (supports advanced expressions)  
    - **Edge Cases:**  
      - Missing description field  
      - Invalid or malformed issue ID causing broken URL  
    - **Sticky Note Context:** Details field extraction and URL creation for AI and messaging.  

#### 2.4 AI Processing

- **Overview:**  
  Uses GPT-4 via LangChain agent to generate a structured summary and IT expert recommendation from the ticket description.

- **Nodes Involved:**  
  - AI Agent - Description Summary  
  - OpenAI Chat Model (used internally by AI Agent)

- **Node Details:**

  - **AI Agent - Description Summary**  
    - **Type:** LangChain Agent node (AI text generation)  
    - **Role:** Summarizes the ticket description and recommends solutions using GPT-4.  
    - **Configuration:**  
      - Text input: `{{$json.description}}` (from Set node)  
      - System message prompt: Custom instructions to generate a structured HTML summary with sections: Main issue, Priority, Recommended solution (acting as IT expert)  
      - Prompt type: Define (custom prompt)  
      - Uses OpenAI GPT-4o-mini model via linked OpenAI Chat Model node  
    - **Input/Output:**  
      - Input: Description text  
      - Output: AI-generated HTML summary  
    - **Version Requirements:** Requires n8n LangChain integration v2+ and OpenAI API access  
    - **Edge Cases:**  
      - API rate limits or timeouts  
      - Unexpected AI response format (non-HTML or incomplete)  
      - Empty or overly short descriptions causing poor summaries  
    - **Sub-workflow:** Uses OpenAI Chat Model node for language model inference  

  - **OpenAI Chat Model**  
    - **Type:** LangChain OpenAI Chat Model node  
    - **Role:** Provides GPT-4o-mini model inference to the AI Agent node  
    - **Configuration:**  
      - Model: GPT-4o-mini (GPT-4 optimized mini)  
      - Credentials: OpenAI API key  
    - **Input/Output:**  
      - Input: Prompt text from AI Agent  
      - Output: AI-generated text response  
    - **Edge Cases:**  
      - API authentication errors  
      - Model unavailability or quota exceeded  

#### 2.5 Output Delivery

- **Overview:**  
  Sends a richly formatted message including ticket details and AI summary to a dedicated Microsoft Teams channel.

- **Nodes Involved:**  
  - MS Teams message to Support channel

- **Node Details:**

  - **MS Teams message to Support channel**  
    - **Type:** Microsoft Teams node (channel message)  
    - **Role:** Posts an HTML-formatted message with new ticket details and AI summary to a specified Teams channel.  
    - **Configuration:**  
      - Resource: Channel Message  
      - Operation: Create  
      - Team ID: Pre-configured (e.g., Support Team)  
      - Channel ID: Pre-configured (e.g., General channel)  
      - Message content: HTML with dynamic placeholders for subject, ticket ID (linked URL), priority, and AI summary  
      - Credentials: Microsoft Teams OAuth2 with appropriate permissions  
    - **Input/Output:**  
      - Input: Data from AI Agent and Set node  
      - Output: Confirmation of message posted  
    - **Version Requirements:** Requires Microsoft Teams node v2+ and OAuth2 credentials  
    - **Edge Cases:**  
      - OAuth token expiration or permission issues  
      - Message formatting errors (invalid HTML)  
      - Network or API downtime  
    - **Sticky Note Context:** Advises use of technical user and channel selection for message posting.  

---

### 3. Summary Table

| Node Name                         | Node Type                             | Functional Role                                   | Input Node(s)                   | Output Node(s)                   | Sticky Note                                                                                                    |
|----------------------------------|-------------------------------------|-------------------------------------------------|--------------------------------|---------------------------------|---------------------------------------------------------------------------------------------------------------|
| Catch Easy Webhook - New Issue Created | Webhook                             | Entry point for new ticket creation events       | -                              | Get a new ticket by ID           | ## Helpdesk Ticket Notification to MS Teams: explains webhook setup and overall workflow purpose              |
| Get a new ticket by ID            | Easy Redmine API node                | Retrieves detailed ticket info by ID             | Catch Easy Webhook - New Issue Created | Pick Description & Create URL to issue | Notes requirement for technical user API credentials and reliance on saved filter                              |
| Pick Description & Create URL to issue | Set node                          | Extracts description and constructs ticket URL   | Get a new ticket by ID          | AI Agent - Description Summary   | Describes field extraction and URL construction for AI input and Teams message                                |
| AI Agent - Description Summary    | LangChain Agent                     | Summarizes description and recommends solution   | Pick Description & Create URL to issue | MS Teams message to Support channel | Highlights AI summarization with GPT-4 and structured HTML output                                            |
| OpenAI Chat Model                 | LangChain OpenAI Chat Model          | Provides GPT-4 model inference                    | AI Agent - Description Summary  | AI Agent - Description Summary   | Underlying GPT-4 model used for AI summarization                                                              |
| MS Teams message to Support channel | Microsoft Teams node                | Sends formatted message to Teams channel          | AI Agent - Description Summary  | -                               | Advises use of technical user with permissions, message formatting, and channel selection                      |
| Sticky Note2                     | Sticky Note                         | Workflow overview and step summary                | -                              | -                               | ## Call for issues in Easy Redmine & send relevant information to MS Teams channel: describes workflow steps  |
| Sticky Note3                     | Sticky Note                         | Demonstrates final output message format          | -                              | -                               | Shows example of final Teams message including AI summary                                                     |
| Sticky Note4                     | Sticky Note                         | Detailed workflow explanation and usage notes     | -                              | -                               | Extensive notes on workflow purpose, use cases, setup instructions, requirements, and support contacts        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: Unique identifier (e.g., `cc5d8044-9921-4b50-8ad4-9a2f1b6718ec`)  
   - Purpose: Receive new ticket creation events from Easy Redmine  

2. **Add Easy Redmine Node**  
   - Type: Easy Redmine (Get One Issue)  
   - Operation: Get One  
   - Parameter: `id` = `{{$json.body.issue.id}}` (from webhook)  
   - Credentials: Configure Easy Redmine API with technical user credentials  
   - Connect input from Webhook node output  

3. **Add Set Node for Data Preparation**  
   - Type: Set  
   - Create fields:  
     - `description` = `{{$json.issue.description}}`  
     - `url link to issue` = `"https://easyredmine-application.com/issues/{{$json.issue.id}}"`  
   - Connect input from Easy Redmine node output  

4. **Add LangChain AI Agent Node**  
   - Type: LangChain Agent  
   - Text input: `{{$json.description}}` (from Set node)  
   - System message prompt:  
     ```
     You are a helpful assistant who summarises read descriptions of the task provided and creates a summary of the task description in form:
     Main issue: 
     Priority of the issue:
     Recommend possible solution:
     When answering the "recommended possible solution" act as an IT expert.
     
     Return response in HTML structure.
     ```  
   - Prompt type: Define  
   - Connect input from Set node output  

5. **Add OpenAI Chat Model Node**  
   - Type: LangChain OpenAI Chat Model  
   - Model: GPT-4o-mini (or GPT-4 equivalent)  
   - Credentials: OpenAI API key with access to GPT-4 or equivalent  
   - Connect AI Agent node's language model input to this node  

6. **Add Microsoft Teams Node**  
   - Type: Microsoft Teams  
   - Operation: Create Channel Message  
   - Set Team ID and Channel ID to target Teams channel (e.g., Support Team / General)  
   - Message content (HTML) example:  
     ```
     <b>❗️New ticket was created:</b><br><br>
     <b>Subject:</b> {{ $('Get a new ticket by ID').item.json.issue.subject }}<br>
     <b><a href="{{ $('Pick Description & Create URL to issue').item.json['url link to issue'] }}">Ticket ID:</a></b> {{ $('Get a new ticket by ID').item.json.issue.id }}<br>
     <b>Priority:</b> {{ $('Get a new ticket by ID').item.json.issue.priority.name }}<br>
     <b>AI summary:</b> {{ $json.output }}
     ```  
   - Credentials: Microsoft Teams OAuth2 with permissions to post messages  
   - Connect input from AI Agent node output  

7. **Final Connections:**  
   - Connect nodes in order:  
     Webhook → Easy Redmine → Set → AI Agent (text input) → Microsoft Teams  
   - Connect AI Agent's language model input to OpenAI Chat Model node  

8. **Additional Setup:**  
   - Configure Easy Redmine webhook on the Easy Redmine platform to POST to the n8n webhook URL  
   - Ensure all credentials are valid and have appropriate API permissions  
   - Test workflow with sample ticket creation  

---

### 5. General Notes & Resources

| Note Content                                                                                                               | Context or Link                                                                                     |
|----------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| Workflow sends helpdesk ticket alerts to Microsoft Teams with GPT-4 summaries to improve team responsiveness and clarity. | Overall project purpose                                                                             |
| Setup requires Easy Redmine application with saved filters and webhook configured to trigger n8n webhook POST endpoint.    | Easy Redmine webhook setup: https://easy-redmine-application.com/easy_web_hooks                    |
| Microsoft Teams requires OAuth2 credentials with permissions to post messages in the target team and channel.              | MS Teams OAuth setup                                                                              |
| AI summarization uses GPT-4 via LangChain integration in n8n; ensure OpenAI API quota and token availability.               | OpenAI LangChain node docs: https://docs.n8n.io/integrations/builtin/nodes/n8n-nodes-langchain/   |
| For help or support, visit n8n community or Easy8.ai YouTube channel.                                                       | n8n community: https://community.n8n.io/u/easy8.ai; YouTube: https://www.youtube.com/@easy8ai      |

---

**Disclaimer:**  
The text provided is derived exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.