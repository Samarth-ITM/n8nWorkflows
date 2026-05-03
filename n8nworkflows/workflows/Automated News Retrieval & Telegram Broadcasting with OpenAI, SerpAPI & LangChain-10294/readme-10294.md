Automated News Retrieval & Telegram Broadcasting with OpenAI, SerpAPI & LangChain

https://n8nworkflows.xyz/workflows/automated-news-retrieval---telegram-broadcasting-with-openai--serpapi---langchain-10294


# Automated News Retrieval & Telegram Broadcasting with OpenAI, SerpAPI & LangChain

### 1. Workflow Overview

This workflow automates the retrieval of news content and broadcasts it to Telegram channels using advanced AI processing and multiple integrated APIs. It is designed for users who want to schedule news updates, query external sources, process the information with AI agents, and distribute summaries or alerts through Telegram and email.

The workflow is logically divided into these blocks:

- **1.1 Trigger and Input Reception:** Scheduling and user form submission trigger the workflow.
- **1.2 Preprocessing and Field Editing:** Standardizing and preparing incoming data for AI processing.
- **1.3 AI Processing and Memory:** Utilizing LangChain-based AI agents with memory buffers and multiple AI tools (Google Gemini, SerpAPI, HTTP Requests) for content generation and decision-making.
- **1.4 Conditional Logic:** Determining whether to proceed with broadcasting or to halt.
- **1.5 Broadcasting and Notifications:** Posting messages on Telegram channels, sending text messages, and emails, including integration with external services (Rapiwa).
- **1.6 Support Nodes:** Utility nodes like wait timers and no-op placeholders.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger and Input Reception

- **Overview:** This block initiates the workflow either by a scheduled trigger or by user form submission.
- **Nodes Involved:**  
  - Schedule  
  - On form submission  

**Node Details:**

- **Schedule**  
  - Type: Schedule Trigger  
  - Role: Periodically triggers the workflow to start automated news retrieval.  
  - Configuration: Default schedule parameters (not explicitly set in JSON).  
  - Inputs: None (trigger node)  
  - Outputs: Connects to "Edit Fields" node.  
  - Edge Cases: If scheduling is misconfigured, workflow won't start automatically.

- **On form submission**  
  - Type: Form Trigger  
  - Role: Starts workflow upon user submission of a web form, allowing manual input or override.  
  - Configuration: Webhook URL enabled for receiving form data.  
  - Inputs: None (trigger node)  
  - Outputs: Connects to "Edit Fields1" node.  
  - Edge Cases: Form data validation failure or webhook connectivity issues.

---

#### 2.2 Preprocessing and Field Editing

- **Overview:** Prepares and sanitizes input data from triggers, setting necessary fields for downstream AI processing.
- **Nodes Involved:**  
  - Edit Fields  
  - Edit Fields1  

**Node Details:**

- **Edit Fields**  
  - Type: Set (Data transformation)  
  - Role: Adjusts or adds fields to the data received from the Schedule trigger.  
  - Configuration: No explicit parameters shown; likely sets or overwrites key fields.  
  - Inputs: From Schedule node.  
  - Outputs: Connects to "AI Agent" node.  
  - Edge Cases: Missing or malformed data may cause AI Agent to receive invalid input.

- **Edit Fields1**  
  - Type: Set (Data transformation)  
  - Role: Similar to "Edit Fields" but processes form submission input.  
  - Configuration: Not explicitly detailed.  
  - Inputs: From On form submission node.  
  - Outputs: Connects to "AI Agent" node.  
  - Edge Cases: Same as above, data integrity is critical.

---

#### 2.3 AI Processing and Memory

- **Overview:** Core processing block where AI agents analyze input, fetch documents, search the web, and generate content using multiple AI tools, supported by memory for context retention.
- **Nodes Involved:**  
  - AI Agent  
  - AI Agent Tool  
  - Simple Memory  
  - Think  
  - Google Gemini Chat Model  
  - SerpAPI  
  - HTTP Request  
  - Get a document in Google Docs  

**Node Details:**

- **AI Agent**  
  - Type: LangChain Agent  
  - Role: Central AI decision-maker that processes input, invokes tools, and routes logic.  
  - Configuration: Uses linked AI tools, memory, and language models.  
  - Inputs: From Edit Fields and Edit Fields1 nodes.  
  - Outputs: Connects to "If" node for conditional branching.  
  - Version: 2.2  
  - Edge Cases: AI model errors, tool invocation failures, timeout, or API quota limits.

- **AI Agent Tool**  
  - Type: LangChain Agent Tool  
  - Role: Provides AI Agent with external tool capabilities (e.g., web search, document retrieval).  
  - Configuration: Connected to multiple tools like SerpAPI, HTTP Request, Google Docs.  
  - Inputs: From Think, SerpAPI, HTTP Request, Get a document in Google Docs, Google Gemini Chat Model nodes, feeding into AI Agent.  
  - Outputs: Feeds back into AI Agent.  
  - Edge Cases: API authentication errors, network issues, parsing errors.

- **Simple Memory**  
  - Type: LangChain Memory Buffer Window  
  - Role: Maintains conversation or processing context for AI Agent.  
  - Inputs: Linked to AI Agent.  
  - Outputs: Feeds into AI Agent.  
  - Edge Cases: Memory overflow or context length limits affecting AI responses.

- **Think**  
  - Type: LangChain Tool Think  
  - Role: An intermediate AI tool for reasoning or planning steps.  
  - Inputs: Feeds AI Agent Tool.  
  - Outputs: Feeds AI Agent.  
  - Edge Cases: AI reasoning failures or stalled processes.

- **Google Gemini Chat Model**  
  - Type: LangChain Language Model (Google Gemini)  
  - Role: Provides chat-based language model capabilities for content generation.  
  - Inputs: Feeds AI Agent Tool and AI Agent.  
  - Outputs: Feeds AI Agent Tool and AI Agent.  
  - Edge Cases: API limits, authentication errors, latency.

- **SerpAPI**  
  - Type: LangChain Tool SerpApi  
  - Role: Performs web search queries to retrieve news or relevant data.  
  - Inputs: Feeds AI Agent Tool.  
  - Outputs: Feeds AI Agent Tool.  
  - Edge Cases: API quota exceeded, invalid search queries, rate limiting.

- **HTTP Request**  
  - Type: HTTP Request Tool  
  - Role: Makes general HTTP requests to fetch external data or APIs.  
  - Inputs: Feeds AI Agent Tool.  
  - Outputs: Feeds AI Agent Tool.  
  - Edge Cases: Timeout, network errors, invalid responses.

- **Get a document in Google Docs**  
  - Type: Google Docs Tool  
  - Role: Retrieves documents from Google Docs to provide content or context.  
  - Inputs: Feeds AI Agent Tool.  
  - Outputs: Feeds AI Agent Tool.  
  - Edge Cases: Authentication errors, missing document, permission issues.

---

#### 2.4 Conditional Logic

- **Overview:** Decides whether to proceed with broadcasting or to stop based on AI Agent output.
- **Nodes Involved:**  
  - If  

**Node Details:**

- **If**  
  - Type: Conditional Node (If)  
  - Role: Checks conditions on AI Agent output to branch flow.  
  - Configuration: Not detailed, but presumably tests if news content or alerts are valid and require broadcasting.  
  - Inputs: From AI Agent.  
  - Outputs:  
    - True branch: Connects to "Wait" node.  
    - False branch: Connects to "Do nothing" node.  
  - Edge Cases: Incorrect condition leading to missed broadcasts or false positives.

---

#### 2.5 Broadcasting and Notifications

- **Overview:** Handles message posting to Telegram channels, sending Telegram text messages, emails, and invoking external notification services.
- **Nodes Involved:**  
  - Wait  
  - Post on Channel  
  - Send a text message  
  - Rapiwa  
  - Send a message  

**Node Details:**

- **Wait**  
  - Type: Wait Node  
  - Role: Introduces delay before broadcasting to manage timing or rate limits.  
  - Configuration: Standard wait (duration not specified).  
  - Inputs: From If node (true branch).  
  - Outputs: Connects to "Post on Channel" and "Send a text message".  
  - Edge Cases: Overlong wait causing delays in timely broadcast.

- **Post on Channel**  
  - Type: Telegram Node  
  - Role: Posts message to a Telegram channel.  
  - Configuration: Uses Telegram bot credentials, configured to post messages (chat id and content set dynamically).  
  - Inputs: From Wait node.  
  - Outputs: Connects to "Rapiwa" and "Send a message".  
  - Edge Cases: Telegram API errors, invalid chat IDs, bot permissions.

- **Send a text message**  
  - Type: Telegram Node  
  - Role: Sends direct Telegram messages (e.g., to users or groups).  
  - Configuration: Uses Telegram bot credentials, parameters for recipient and message content configured dynamically.  
  - Inputs: From Wait node.  
  - Outputs: None (end node).  
  - Edge Cases: Same as Post on Channel.

- **Rapiwa**  
  - Type: Rapiwa Node (third-party notification service)  
  - Role: Sends notifications or performs external API calls related to broadcasting.  
  - Configuration: Requires Rapiwa API credentials.  
  - Inputs: From Post on Channel.  
  - Outputs: None (end node).  
  - Edge Cases: API limits, authentication failure.

- **Send a message**  
  - Type: Gmail Node  
  - Role: Sends email notifications or reports related to the broadcast.  
  - Configuration: Uses Gmail OAuth2 credentials, configured with recipient, subject, and content.  
  - Inputs: From Post on Channel.  
  - Outputs: None (end node).  
  - Edge Cases: Gmail API quota, invalid email addresses, authentication errors.

---

#### 2.6 Support Nodes

- **Overview:** Utility nodes supporting workflow control and debugging.
- **Nodes Involved:**  
  - Do nothing  
  - Sticky Notes (disabled)  

**Node Details:**

- **Do nothing**  
  - Type: No Operation Node  
  - Role: Ends flow on false condition branch without action.  
  - Inputs: From If node (false branch).  
  - Outputs: None.

- **Sticky Notes (Sticky Note5, Sticky Note6, Sticky Note7, Sticky Note9)**  
  - Type: Sticky Note  
  - Role: Disabled notes presumably used for documentation or comments, but empty in this workflow.  
  - Inputs/Outputs: None.  
  - Edge Cases: None.

---

### 3. Summary Table

| Node Name                 | Node Type                          | Functional Role                        | Input Node(s)                 | Output Node(s)                       | Sticky Note |
|---------------------------|----------------------------------|-------------------------------------|------------------------------|------------------------------------|-------------|
| Schedule                  | Schedule Trigger                 | Periodic workflow start              | None                         | Edit Fields                        |             |
| On form submission        | Form Trigger                    | Manual input trigger                 | None                         | Edit Fields1                      |             |
| Edit Fields               | Set                             | Prepares data from Schedule          | Schedule                     | AI Agent                         |             |
| Edit Fields1              | Set                             | Prepares data from form submission   | On form submission           | AI Agent                         |             |
| AI Agent                  | LangChain Agent                 | Core AI processing                   | Edit Fields, Edit Fields1    | If                              |             |
| AI Agent Tool             | LangChain Agent Tool            | Provides AI external tools           | Think, SerpAPI, HTTP Request, Google Docs, Google Gemini Chat Model | AI Agent                         |             |
| Simple Memory             | LangChain Memory Buffer Window  | Context memory for AI Agent          | AI Agent                     | AI Agent                         |             |
| Think                     | LangChain Tool Think            | Intermediate reasoning step          | AI Agent Tool                | AI Agent                         |             |
| Google Gemini Chat Model  | LangChain Language Model        | AI language generation               | AI Agent Tool, AI Agent      | AI Agent Tool, AI Agent          |             |
| SerpAPI                   | LangChain Tool SerpApi          | Web search for news and data         | AI Agent Tool                | AI Agent Tool                    |             |
| HTTP Request              | HTTP Request Tool               | External API calls                   | AI Agent Tool                | AI Agent Tool                    |             |
| Get a document in Google Docs | Google Docs Tool             | Retrieve Google Docs documents       | AI Agent Tool                | AI Agent Tool                    |             |
| If                        | Conditional                    | Branch based on AI output             | AI Agent                     | Wait (true), Do nothing (false) |             |
| Wait                      | Wait                          | Delay before broadcasting             | If (true)                   | Post on Channel, Send a text message |             |
| Post on Channel           | Telegram                       | Post message on Telegram channel     | Wait                         | Rapiwa, Send a message           |             |
| Send a text message       | Telegram                       | Send direct Telegram text message    | Wait                         | None                           |             |
| Rapiwa                    | Rapiwa                        | External notification service         | Post on Channel              | None                           |             |
| Send a message            | Gmail                         | Send email notifications             | Post on Channel              | None                           |             |
| Do nothing                | No Operation                  | Ends workflow without action         | If (false)                  | None                           |             |
| Sticky Note5              | Sticky Note                   | Disabled, no content                  | None                         | None                           |             |
| Sticky Note6              | Sticky Note                   | Disabled, no content                  | None                         | None                           |             |
| Sticky Note7              | Sticky Note                   | Disabled, no content                  | None                         | None                           |             |
| Sticky Note9              | Sticky Note                   | Disabled, no content                  | None                         | None                           |             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Schedule Trigger node:**  
   - Type: Schedule Trigger  
   - Set desired frequency for automated news retrieval (e.g., daily at 8:00 AM).

2. **Create the On form submission node:**  
   - Type: Form Trigger  
   - Configure webhook URL and form fields for manual news retrieval input.

3. **Create two Set nodes for field editing:**  
   - Names: "Edit Fields" and "Edit Fields1"  
   - Configure each to set or overwrite necessary fields from Schedule and form inputs respectively (e.g., keywords, topics).  
   - Connect Schedule → Edit Fields and On form submission → Edit Fields1.

4. **Create the AI Agent node (LangChain Agent):**  
   - Type: LangChain agent (version 2.2)  
   - Configure with access to AI language models and tools (see below).  
   - Connect both Edit Fields and Edit Fields1 nodes to this AI Agent.

5. **Set up AI tools for the AI Agent:**

   - **AI Agent Tool node:**  
     - Connect to the AI Agent's "ai_tool" input.  
     - Configure with linked tools:
     
     - **SerpAPI node:**  
       - Type: LangChain SerpApi tool  
       - Configure API key for SerpAPI.  
       - Connect to AI Agent Tool node.
     
     - **HTTP Request node:**  
       - Type: HTTP Request Tool  
       - Configure for external API calls if needed.  
       - Connect to AI Agent Tool node.
     
     - **Get a document in Google Docs node:**  
       - Type: Google Docs Tool  
       - Authenticate with Google OAuth2 credentials, enable access to required Docs.  
       - Connect to AI Agent Tool node.
     
     - **Google Gemini Chat Model node:**  
       - Type: LangChain Language Model  
       - Configure Google Gemini API credentials.  
       - Connect to both AI Agent Tool and AI Agent nodes.

6. **Add Simple Memory node:**  
   - Type: LangChain Memory Buffer Window  
   - Connect memory input and output to AI Agent node to maintain context.

7. **Add Think node (LangChain Tool Think):**  
   - Connect to AI Agent Tool node, then to AI Agent node to assist reasoning.

8. **Create If node for conditional branching:**  
   - Connect AI Agent output to If node.  
   - Define condition(s) based on AI Agent output to determine if broadcasting should proceed.

9. **Create Wait node:**  
   - Connect If node's true branch to Wait node.  
   - Configure delay duration as desired.

10. **Create Telegram nodes for broadcasting:**

    - **Post on Channel:**  
      - Type: Telegram  
      - Configure Telegram bot credentials (Bot Token).  
      - Set parameters to post message to the target Telegram channel.  
      - Connect Wait node to Post on Channel.
      
    - **Send a text message:**  
      - Type: Telegram  
      - Configure Telegram bot credentials.  
      - Set parameters for sending direct messages.  
      - Connect Wait node to Send a text message.

11. **Create Rapiwa node:**  
    - Configure with Rapiwa API credentials.  
    - Connect Post on Channel node to Rapiwa node.

12. **Create Send a message node (Gmail):**  
    - Type: Gmail node  
    - Configure OAuth2 credentials for Gmail.  
    - Set recipient, subject, and message content dynamically.  
    - Connect Post on Channel node to Send a message node.

13. **Create Do nothing node:**  
    - Type: No Operation (NoOp)  
    - Connect If node's false branch to Do nothing node.

14. **Disable or omit Sticky Notes:**  
    - These are optional and contain no content.

---

### 5. General Notes & Resources

| Note Content                                                                                       | Context or Link                                                                                   |
|--------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| For Telegram integration, ensure your bot has necessary permissions to post messages and send texts. | Telegram Bot API documentation: https://core.telegram.org/bots/api                               |
| LangChain nodes require setup of credentials for OpenAI, Google Gemini, SerpAPI, Google Docs, etc. | Refer to n8n LangChain node documentation for configuring credentials and API access.           |
| Rapiwa is a third-party notification service; obtain API keys and verify usage limits.            | Rapiwa official site: https://rapiwa.com                                                          |
| Use environment variables or n8n credentials for managing sensitive API keys securely.           | n8n credentials setup guide: https://docs.n8n.io/credentials/                                    |
| Testing the workflow with sample inputs is recommended to handle edge cases before production.   |                                                                                                |

---

_Disclaimer: The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to content policies and contains no illegal, offensive, or protected elements. All handled data are legal and public._