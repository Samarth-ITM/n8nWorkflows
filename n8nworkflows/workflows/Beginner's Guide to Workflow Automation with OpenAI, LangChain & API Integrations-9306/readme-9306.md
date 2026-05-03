Beginner's Guide to Workflow Automation with OpenAI, LangChain & API Integrations

https://n8nworkflows.xyz/workflows/beginner-s-guide-to-workflow-automation-with-openai--langchain---api-integrations-9306


# Beginner's Guide to Workflow Automation with OpenAI, LangChain & API Integrations

---

### 1. Workflow Overview

This workflow, titled **"Beginner's Guide to Workflow Automation with OpenAI, LangChain & API Integrations"**, serves as a comprehensive demonstration for newcomers to n8n. Its primary purpose is to showcase foundational concepts of workflow automation, including triggers, data manipulation, conditional branching, external API calls, and integration with AI services such as OpenAI and LangChain. The workflow is structured into several logical blocks reflecting these core functions:

- **1.1 Manual Trigger & Data Preparation:** Starting point with manual input and initial data setup.
- **1.2 Data Processing & Conditional Routing:** Custom JavaScript processing and routing data based on conditions.
- **1.3 API Integration & Data Enrichment:** Fetching external user data, splitting arrays, extracting fields, and enriching with AI-generated content.
- **1.4 Webhook Reception & Response:** Handling incoming HTTP requests with webhook and responding accordingly.
- **1.5 AI Chatbot with LangChain:** Interactive AI assistant powered by LangChain with memory, tools, and OpenAI language models.
- **1.6 Scheduled API Calls:** Demonstrating scheduled triggers to automate periodic tasks.
- **1.7 Final Summary & Merge:** Combining data branches and summarizing workflow execution.

---

### 2. Block-by-Block Analysis

#### 1.1 Manual Trigger & Data Preparation

**Overview:**  
This block initiates the workflow manually and sets up initial data for processing.

**Nodes Involved:**  
- Start Here - Manual Trigger  
- Set Initial Data  
- Welcome Note (sticky)  
- About Set Node (sticky)

**Node Details:**  

- **Start Here - Manual Trigger**  
  - *Type:* Manual Trigger  
  - *Role:* Entry point allowing manual execution of the workflow.  
  - *Config:* Default, no parameters.  
  - *Input:* None  
  - *Output:* Triggers "Set Initial Data" node.  
  - *Potential Failures:* None typical; relies on user to start workflow.

- **Set Initial Data**  
  - *Type:* Set  
  - *Role:* Creates or modifies data fields to initialize the dataset.  
  - *Config:* Empty options; designed to accept input or static data in future modifications.  
  - *Input:* Trigger from manual start.  
  - *Output:* Sends data to "Code Node - Custom Processing."  
  - *Expressions:* Supports dynamic expressions (highlighted in sticky note).  
  - *Failures:* Expression errors if fields are missing or syntax is incorrect.

- **Welcome Note (Sticky Note)**  
  - *Purpose:* Documentation introducing the workflow's learning goals and structure.  
  - *Sticky Content:* Explains triggers, data processing, conditionals, API calls, AI agents.

- **About Set Node (Sticky Note)**  
  - *Purpose:* Describes the functionality of the Set node, including data field creation and use of expressions.

---

#### 1.2 Data Processing & Conditional Routing

**Overview:**  
Processes initial data with custom JavaScript and uses an IF node to route data based on a numeric condition.

**Nodes Involved:**  
- Code Node - Custom Processing  
- IF Condition - Route Data  
- True Branch Data  
- False Branch Data  
- Merge Both Branches  
- About Code Node (sticky)  
- About IF Node (sticky)  
- About Merge Node (sticky)

**Node Details:**  

- **Code Node - Custom Processing**  
  - *Type:* Code  
  - *Role:* Runs custom JavaScript to manipulate and enrich data items.  
  - *Config:* Multiplies a sample number by 2, calculates message length, timestamps processing, manipulates strings, and adds tags.  
  - *Input:* Data from "Set Initial Data."  
  - *Output:* Sends processed data to "IF Condition - Route Data."  
  - *Potential Failures:* Script syntax errors, undefined fields in input JSON.

- **IF Condition - Route Data**  
  - *Type:* IF  
  - *Role:* Splits data flow based on whether `sample_number` is less than 50.  
  - *Config:* Condition: `$json.sample_number < 50` (strict type validation).  
  - *Input:* Processed data from Code Node.  
  - *Outputs:*  
    - True branch to "True Branch Data" node.  
    - False branch to "False Branch Data" node.  
  - *Failures:* Missing or non-numeric `sample_number` field, expression errors.

- **True Branch Data**  
  - *Type:* Set  
  - *Role:* Placeholder for processing data that meets the IF condition.  
  - *Input:* True output of IF node.  
  - *Output:* Connects to "Merge Both Branches."

- **False Branch Data**  
  - *Type:* Set  
  - *Role:* Placeholder for processing data that does not meet the IF condition.  
  - *Input:* False output of IF node.  
  - *Output:* Connects to "Merge Both Branches."

- **Merge Both Branches**  
  - *Type:* Merge  
  - *Role:* Combines data streams from both branches into a single stream for further processing.  
  - *Config:* Mode is "combine" (pairwise merge).  
  - *Input:* Data from True Branch and False Branch nodes.  
  - *Output:* Sends merged data to "Final Summary."  
  - *Failures:* Mismatched input lengths or data inconsistencies.

- **Sticky Notes:**  
  - *About Code Node:* Explains JavaScript usage, available context, and example patterns.  
  - *About IF Node:* Details conditional logic, output branches, and supported condition types.  
  - *About Merge Node:* Describes different merge modes and use cases.

---

#### 1.3 API Integration & Data Enrichment

**Overview:**  
Fetches user data from an external API, processes it item-by-item, and enriches each user record with AI-generated fun facts about their company.

**Nodes Involved:**  
- HTTP Request - Fetch Users  
- Split Users Array  
- Extract User Fields  
- OpenAI API - Generate Fun Fact  
- About HTTP Request (sticky)  
- About Split Out (sticky)  
- About OpenAI (sticky)

**Node Details:**  

- **HTTP Request - Fetch Users**  
  - *Type:* HTTP Request  
  - *Role:* Retrieves user data from JSONPlaceholder API (https://jsonplaceholder.typicode.com/users).  
  - *Config:* GET method, no authentication.  
  - *Input:* None (could be triggered manually or scheduled).  
  - *Output:* JSON array of users to "Split Users Array."  
  - *Failures:* Network errors, API downtime, malformed responses.

- **Split Users Array**  
  - *Type:* Split Out  
  - *Role:* Converts array of users in one item to multiple individual items for per-user processing.  
  - *Config:* Splits on `data` field.  
  - *Input:* Array from HTTP Request node.  
  - *Output:* Sends individual user items to "Extract User Fields."  
  - *Failures:* Missing field to split, empty arrays.

- **Extract User Fields**  
  - *Type:* Set  
  - *Role:* Extracts specific fields from each user JSON: name, email, and company name (nested).  
  - *Config:* Assigns `user_name`, `user_email`, and `company` using expressions accessing input JSON fields.  
  - *Input:* Individual user items.  
  - *Output:* Sends simplified user data to OpenAI API node.

- **OpenAI API - Generate Fun Fact**  
  - *Type:* HTTP Request with OpenAI credential  
  - *Role:* Calls OpenAI Chat Completion API (model `gpt-4o-mini`) to generate a fun fact about the user's company.  
  - *Config:*  
    - POST to `https://api.openai.com/v1/chat/completions`.  
    - Message prompt dynamically includes user name and company from previous node.  
    - Temperature set to 0.7 for creativity, max tokens 100.  
    - Header includes Content-Type: application/json.  
  - *Input:* User fields from "Extract User Fields."  
  - *Output:* Enriched data with AI-generated text (fun fact).  
  - *Failures:* Authentication errors, quota limits, API timeouts, invalid prompt construction.

- **Sticky Notes:**  
  - *About HTTP Request:* Explains usage of HTTP node for API calls with various methods and authentication options.  
  - *About Split Out:* Describes purpose of splitting arrays into multiple items for granular processing.  
  - *About OpenAI:* Provides setup instructions for OpenAI integration and key parameters.

---

#### 1.4 Webhook Reception & Response

**Overview:**  
Receives external HTTP requests and responds with a JSON confirmation including received data and timestamp.

**Nodes Involved:**  
- Webhook - Receive Data  
- Send Webhook Response  
- About Webhooks (sticky)

**Node Details:**  

- **Webhook - Receive Data**  
  - *Type:* Webhook  
  - *Role:* HTTP POST endpoint at path `/beginner-webhook` to receive external data.  
  - *Config:* Response mode set to use a response node.  
  - *Input:* External POST requests.  
  - *Output:* Sends received data to "Send Webhook Response."  
  - *Failures:* Incorrect HTTP methods, malformed JSON, network issues.

- **Send Webhook Response**  
  - *Type:* Respond to Webhook  
  - *Role:* Sends JSON response confirming success, echoes received data, and adds a processing timestamp.  
  - *Config:* Response body constructed dynamically using incoming request body and current ISO timestamp.  
  - *Input:* Data from webhook.  
  - *Output:* Final output to HTTP client.  
  - *Failures:* Expression errors if `$json.body` or `$now` is unavailable.

- **Sticky Note:**  
  - *About Webhooks:* Describes webhook purpose, URL structure, use cases, and example curl request.

---

#### 1.5 AI Chatbot with LangChain

**Overview:**  
Implements an AI-powered interactive assistant using LangChain components, including chat trigger, memory buffer, tools for current time and calculations, and the AI agent itself.

**Nodes Involved:**  
- Chat Trigger - AI Assistant  
- AI Agent - n8n Assistant  
- Window Buffer Memory  
- OpenAI Chat Model  
- Tool: Current Time  
- Tool: Calculator  
- About Chat + AI (sticky)  
- About AI Tools (sticky)

**Node Details:**  

- **Chat Trigger - AI Assistant**  
  - *Type:* LangChain Chat Trigger  
  - *Role:* Public webhook endpoint for initiating AI chat sessions.  
  - *Config:* Public access enabled, with title and subtitle describing the assistant.  
  - *Initial Messages:* Friendly greeting and usage examples.  
  - *Output:* Starts AI processing workflow.  
  - *Failures:* Webhook misconfiguration, accessibility issues.

- **AI Agent - n8n Assistant**  
  - *Type:* LangChain Agent  
  - *Role:* Core AI processor that interprets user inputs and decides on actions.  
  - *Config:*  
    - System message defines assistant role for beginners, teaching nodes and best practices.  
    - Output parser enabled to format responses.  
  - *Input:* From Chat Trigger and LangChain components.  
  - *Output:* Final AI responses to chat clients.  
  - *Failures:* Model errors, misconfigured system prompts.

- **Window Buffer Memory**  
  - *Type:* LangChain Memory Buffer Window  
  - *Role:* Stores last 10 messages to provide context to AI agent.  
  - *Input:* Connected to AI Agent for memory management.  
  - *Output:* Feeds memory context to AI processing.  
  - *Failures:* Memory overflow or loss if improperly configured.

- **OpenAI Chat Model**  
  - *Type:* LangChain OpenAI Chat Model  
  - *Role:* Language model provider for AI agent.  
  - *Config:* max tokens 500, temperature 0.7 for balanced creativity.  
  - *Input:* Messages and memory context.  
  - *Output:* Language model completions to AI agent.  
  - *Failures:* API limits, credential errors.

- **Tool: Current Time**  
  - *Type:* LangChain Code Tool  
  - *Role:* Returns current date/time when requested by user.  
  - *Config:* Custom tool named `get_current_time` with descriptive usage.  
  - *Input:* AI agent queries.  
  - *Output:* Current time string.  
  - *Failures:* System clock issues unlikely.

- **Tool: Calculator**  
  - *Type:* LangChain Code Tool  
  - *Role:* Performs simple math calculations from string expressions.  
  - *Config:* JavaScript code executes `eval` on sanitized input expressions.  
  - *Input:* Mathematical expressions as strings.  
  - *Output:* JSON string with result or error message.  
  - *Failures:* Invalid expressions, security risks with eval (noted to replace in production).

- **Sticky Notes:**  
  - *About Chat + AI:* Explains components and use cases of AI chatbots.  
  - *About AI Tools:* Describes different tool types AI agents can use and how they interact.

---

#### 1.6 Scheduled API Calls

**Overview:**  
Demonstrates a scheduled trigger that could initiate periodic workflow executions, such as fetching data on Mondays at 9 AM.

**Nodes Involved:**  
- Schedule - Every Monday 9 AM  
- About Schedules (sticky)

**Node Details:**  

- **Schedule - Every Monday 9 AM**  
  - *Type:* Schedule Trigger  
  - *Role:* Automatically triggers workflow every Monday at 9:00 AM via cron expression `0 9 * * 1`.  
  - *Config:* Cron expression used for precise scheduling.  
  - *Input:* None (time-based trigger).  
  - *Output:* Can be connected to any node requiring periodic execution.  
  - *Failures:* Incorrect cron format, timezone misalignment.

- **Sticky Note:**  
  - *About Schedules:* Details scheduling options, cron examples, and typical use cases.

---

#### 1.7 Final Summary & Merge

**Overview:**  
Concludes the workflow by merging all data branches and providing a summarized report of the execution.

**Nodes Involved:**  
- Final Summary  
- Merge Both Branches (also in 1.2)

**Node Details:**  

- **Final Summary**  
  - *Type:* Set  
  - *Role:* Generates a textual summary of the workflow execution including count of processed items, timestamp, and learning points.  
  - *Config:* Uses expressions to count items processed and current time in ISO format.  
  - *Input:* Merged data stream.  
  - *Output:* Final output, suitable for logs or notifications.  
  - *Failures:* Expression errors if `$items()` or `$now` is unavailable.

---

### 3. Summary Table

| Node Name                   | Node Type                        | Functional Role                           | Input Node(s)                    | Output Node(s)                  | Sticky Note                                                                                          |
|-----------------------------|---------------------------------|-----------------------------------------|---------------------------------|-------------------------------|----------------------------------------------------------------------------------------------------|
| Start Here - Manual Trigger | Manual Trigger                  | Manual workflow entry point              | None                            | Set Initial Data                |                                                                                                    |
| Set Initial Data             | Set                             | Initialize or modify data fields         | Start Here - Manual Trigger     | Code Node - Custom Processing  | About Set Node: Purpose, expressions, dynamic values                                               |
| Welcome Note                | Sticky Note                    | Documentation introduction                | None                            | None                          | Overview of workflow learning goals                                                               |
| About Set Node              | Sticky Note                    | Documentation on Set node usage           | None                            | None                          | Details on data field setting and expressions                                                     |
| Code Node - Custom Processing| Code                            | Custom JavaScript data processing         | Set Initial Data                | IF Condition - Route Data      | About Code Node: JS context, usage patterns                                                       |
| IF Condition - Route Data   | IF                              | Conditional routing based on sample_number| Code Node - Custom Processing   | True Branch Data, False Branch Data| About IF Node: Conditional logic, outputs                                                      |
| True Branch Data            | Set                             | Handles data when IF condition is true   | IF Condition - Route Data (True)| Merge Both Branches            |                                                                                                    |
| False Branch Data           | Set                             | Handles data when IF condition is false  | IF Condition - Route Data (False)| Merge Both Branches           |                                                                                                    |
| Merge Both Branches         | Merge                           | Combines true and false branches          | True Branch Data, False Branch Data| Final Summary                | About Merge Node: Merge modes and use cases                                                      |
| HTTP Request - Fetch Users  | HTTP Request                   | Fetch external user data                   | None                           | Split Users Array              | About HTTP Request: API call methods, auth, examples                                             |
| Split Users Array           | Split Out                      | Split array into individual items          | HTTP Request - Fetch Users      | Extract User Fields            | About Split Out: Convert arrays to items                                                         |
| Extract User Fields         | Set                             | Extract user name, email, company fields  | Split Users Array               | OpenAI API - Generate Fun Fact|                                                                                                    |
| OpenAI API - Generate Fun Fact| HTTP Request (OpenAI)          | Generate AI fun fact about user companies | Extract User Fields             | None                          | About OpenAI: Setup and parameters                                                                |
| Webhook - Receive Data      | Webhook                        | Receive external HTTP POST data            | None                           | Send Webhook Response          | About Webhooks: Purpose, URL, example curl                                                       |
| Send Webhook Response       | Respond to Webhook             | Send JSON response confirming reception   | Webhook - Receive Data          | None                          |                                                                                                    |
| Chat Trigger - AI Assistant | LangChain Chat Trigger         | Entry point for AI chat sessions           | None                           | AI Agent - n8n Assistant       | About Chat + AI: Chatbot components and use cases                                                |
| AI Agent - n8n Assistant    | LangChain Agent                | Processes user input and manages AI logic | Chat Trigger - AI Assistant, Window Buffer Memory, OpenAI Chat Model, Tools | None                          | About AI Tools: Tool types and workflows                                                         |
| Window Buffer Memory        | LangChain Memory Buffer Window | Maintains conversational context          | None                           | AI Agent - n8n Assistant       |                                                                                                    |
| OpenAI Chat Model           | LangChain LM Chat OpenAI       | Provides AI language completions            | None                           | AI Agent - n8n Assistant       |                                                                                                    |
| Tool: Current Time          | LangChain Tool Code            | Returns current date and time               | None                           | AI Agent - n8n Assistant       |                                                                                                    |
| Tool: Calculator            | LangChain Tool Code            | Performs math calculations                   | None                           | AI Agent - n8n Assistant       |                                                                                                    |
| Schedule - Every Monday 9 AM| Schedule Trigger               | Automatically triggers workflow weekly      | None                           | None (not connected)           | About Schedules: Cron usage and examples                                                        |
| Final Summary               | Set                             | Summarizes workflow execution               | Merge Both Branches             | None                          |                                                                                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Manual Trigger Node**  
   - Create a **Manual Trigger** node named "Start Here - Manual Trigger."  
   - No parameters needed.

2. **Set Initial Data Node**  
   - Add a **Set** node named "Set Initial Data."  
   - Leave fields empty for now or add sample data as needed.  
   - Connect output of "Start Here - Manual Trigger" to this node.

3. **Code Node - Custom Processing**  
   - Create a **Code** node named "Code Node - Custom Processing."  
   - Paste JavaScript code that processes each item: doubles a sample number, calculates message length, timestamps processing, converts message to uppercase, adds tags array.  
   - Connect output of "Set Initial Data" to this node.

4. **IF Condition Node**  
   - Add an **IF** node named "IF Condition - Route Data."  
   - Set condition to check if `sample_number < 50` (number comparison, strict).  
   - Connect output of "Code Node - Custom Processing" to this node.

5. **True Branch and False Branch Data Nodes**  
   - Create two **Set** nodes named "True Branch Data" and "False Branch Data."  
   - Configure as placeholders or add relevant data modifications.  
   - Connect the True output of IF node to "True Branch Data," False output to "False Branch Data."

6. **Merge Node**  
   - Add a **Merge** node named "Merge Both Branches."  
   - Set mode to "Combine."  
   - Connect both "True Branch Data" and "False Branch Data" outputs to this node.

7. **Final Summary Node**  
   - Add a **Set** node named "Final Summary."  
   - Add a string field with expression summarizing workflow execution, count of items, and timestamp.  
   - Connect output of "Merge Both Branches" to this node.

8. **HTTP Request Node**  
   - Add an **HTTP Request** node named "HTTP Request - Fetch Users."  
   - Set URL to `https://jsonplaceholder.typicode.com/users`.  
   - Method: GET.  
   - No authentication.  
   - Can be triggered manually or connected to a schedule trigger.  

9. **Split Out Node**  
   - Add a **Split Out** node named "Split Users Array."  
   - Set field to split on `data` or ensure the HTTP response is assigned to that field.  
   - Connect output of HTTP Request node to this node.

10. **Extract User Fields Node**  
    - Add a **Set** node named "Extract User Fields."  
    - Create fields:  
      - `user_name` = `{{$json.name}}`  
      - `user_email` = `{{$json.email}}`  
      - `company` = `{{$json.company.name}}`  
    - Connect output of "Split Users Array" to this node.

11. **OpenAI API Node**  
    - Add an **HTTP Request** node named "OpenAI API - Generate Fun Fact."  
    - Set URL to OpenAI chat completions endpoint: `https://api.openai.com/v1/chat/completions`.  
    - Method: POST.  
    - Use OpenAI credential configured in n8n.  
    - Body parameters:  
      - `model`: `gpt-4o-mini`  
      - `messages`: JSON array with user prompt referencing `user_name` and `company` fields dynamically.  
      - `temperature`: 0.7  
      - `max_tokens`: 100  
    - Header: Content-Type application/json.  
    - Connect output of "Extract User Fields" to this node.

12. **Webhook Node**  
    - Add a **Webhook** node named "Webhook - Receive Data."  
    - Set HTTP Method to POST.  
    - Set path to `beginner-webhook`.  
    - Set response mode to "Response Node."  

13. **Respond to Webhook Node**  
    - Add a **Respond to Webhook** node named "Send Webhook Response."  
    - Configure to respond with JSON:  
      ```json
      {
        "status": "success",
        "received": {{$json.body}},
        "processed_at": {{$now.toISO()}}
      }
      ```  
    - Connect output of "Webhook - Receive Data" to this node.

14. **LangChain Chat Trigger**  
    - Add a **Chat Trigger (LangChain)** node named "Chat Trigger - AI Assistant."  
    - Set webhook ID `beginner-chat-assistant`.  
    - Enable public access.  
    - Set title and subtitle as per workflow.  
    - Add initial messages with greeting and example questions.

15. **LangChain AI Agent**  
    - Add an **AI Agent (LangChain)** node named "AI Agent - n8n Assistant."  
    - Configure system message to define assistant role for beginners with key topics.  
    - Enable output parser.  
    - Connect output of Chat Trigger node to this node.

16. **Memory Buffer Window**  
    - Add a **Window Buffer Memory (LangChain)** node named "Window Buffer Memory."  
    - Set context window length to 10 messages.  
    - Connect its output to AI Agent node memory input.

17. **OpenAI Chat Model (LangChain)**  
    - Add a **OpenAI Chat Model (LangChain)** node named "OpenAI Chat Model."  
    - Set max tokens to 500 and temperature to 0.7.  
    - Connect output to AI Agent language model input.

18. **Tools:**  
    - Add **Tool: Current Time** (LangChain Tool Code) with name `get_current_time`.  
    - Add **Tool: Calculator** (LangChain Tool Code) with JavaScript code to evaluate simple math expressions safely.  
    - Connect both tools as AI tools input to AI Agent node.

19. **Schedule Trigger**  
    - Add a **Schedule Trigger** node named "Schedule - Every Monday 9 AM."  
    - Set cron expression `0 9 * * 1`.  
    - Connect to any node to automate workflow execution on schedule.

20. **Connect and Test**  
    - Verify all node connections as per above.  
    - Test manual trigger and webhook endpoints.  
    - Ensure OpenAI credentials and API keys are configured correctly.  
    - Confirm AI assistant responds via chat webhook.  
    - Validate error handling for missing or malformed data.

---

### 5. General Notes & Resources

| Note Content                                                                                                                           | Context or Link                                                                                  |
|----------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Welcome sticky note outlines key learning goals including triggers, data processing, conditionals, API calls, and AI agents.           | Workflow introduction documentation.                                                           |
| Use free public APIs like JSONPlaceholder to test HTTP request nodes before integrating real endpoints.                                | HTTP Request sticky note.                                                                       |
| OpenAI integration requires proper credential setup; choose models balancing capability and cost (e.g., gpt-4o-mini for efficiency). | OpenAI sticky note.                                                                             |
| LangChain nodes enable building AI assistants with memory, tools, and language models providing interactive help inside n8n.          | About Chat + AI and AI Tools sticky notes.                                                     |
| Scheduled triggers are powerful for automating routine tasks like reports or data sync. Use cron expressions carefully for timezone.   | Schedule sticky note with cron examples.                                                       |
| For the calculator tool, avoid using `eval` in production; implement a proper math expression parser for safety.                       | Tool: Calculator node notes.                                                                   |
| Webhook nodes provide integration points for external services or forms; test using curl or Postman with JSON payloads.               | About Webhooks sticky note.                                                                    |
| This workflow is ideal as a foundational template for beginners to understand common automation patterns combining AI and API calls.  | Overall workflow purpose and use case.                                                         |

---

**Disclaimer:**  
The text provided is exclusively derived from an automated workflow created with n8n, a workflow automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.

---