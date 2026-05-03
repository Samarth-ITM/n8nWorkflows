Build an MCP Server with Google Calendar and Custom Functions

https://n8nworkflows.xyz/workflows/build-an-mcp-server-with-google-calendar-and-custom-functions-3514


# Build an MCP Server with Google Calendar and Custom Functions

### 1. Workflow Overview

This workflow demonstrates how to build an MCP (Modular Chatbot Protocol) Server and Client architecture in n8n, integrating both native nodes (Google Calendar) and custom functions. It targets users who want to create AI Agents that dynamically connect to multiple MCP Servers exposing different tools without needing to update the client when tools change.

The workflow is logically divided into two main MCP Servers:

- **Google Calendar MCP Server:** Provides calendar event management tools (search, create, update, delete).
- **Custom Functions MCP Server:** Offers text conversion (uppercase/lowercase), random user data generation, and joke retrieval.

An AI Agent node connects to both MCP Clients, which in turn connect to the respective MCP Servers. The workflow also includes memory management and utility nodes to handle AI requests and responses.

Logical blocks:

- **1.1 Input Reception and AI Agent Setup:** Receives chat messages and manages AI memory.
- **1.2 MCP Servers Setup:** Defines two MCP Servers exposing Google Calendar tools and custom functions.
- **1.3 MCP Clients Setup:** Connects the AI Agent to the MCP Servers via MCP Clients.
- **1.4 Custom Function Sub-Workflows:** Implements custom tools for text conversion, random user data, and jokes.
- **1.5 Workflow Control and Routing:** Routes function calls to appropriate sub-workflows or nodes.
- **1.6 Utility and Debugging:** Includes debug helpers and sticky notes for instructions and documentation.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and AI Agent Setup

**Overview:**  
This block handles incoming chat messages, manages AI memory, and processes the AI Agent's core logic.

**Nodes Involved:**  
- When chat message received  
- Simple Memory  
- OpenAI 4o  
- AI Agent  

**Node Details:**

- **When chat message received**  
  - Type: `@n8n/n8n-nodes-langchain.chatTrigger`  
  - Role: Entry point for chat messages triggering the AI Agent.  
  - Config: Default webhook with no special options.  
  - Inputs: External chat messages via webhook.  
  - Outputs: Passes data to AI Agent.  
  - Edge cases: Webhook availability, message format errors.

- **Simple Memory**  
  - Type: `@n8n/n8n-nodes-langchain.memoryBufferWindow`  
  - Role: Maintains conversation context for the AI Agent.  
  - Config: Default buffer window memory.  
  - Inputs: From chat trigger.  
  - Outputs: To AI Agent.  
  - Edge cases: Memory overflow or loss.

- **OpenAI 4o**  
  - Type: `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
  - Role: Language model for AI responses.  
  - Config: Uses GPT-4o model for better handling of calendar requests.  
  - Credentials: OpenAI API configured with `n8n-testing2`.  
  - Inputs: From memory node.  
  - Outputs: To AI Agent.  
  - Edge cases: API rate limits, auth failures, model availability.

- **AI Agent**  
  - Type: `@n8n/n8n-nodes-langchain.agent`  
  - Role: Central AI orchestrator connecting to MCP Clients and managing conversation.  
  - Config: System message includes current datetime dynamically.  
  - Inputs: From OpenAI node and MCP Clients.  
  - Outputs: To MCP Clients and memory.  
  - Version: Requires n8n 1.88.0 or higher.  
  - Edge cases: Expression evaluation errors, connection issues with MCP Clients.

---

#### 2.2 MCP Servers Setup

**Overview:**  
Defines two MCP Servers exposing tools: one for Google Calendar operations and one for custom functions.

**Nodes Involved:**  
- Google Calendar MCP (MCP Trigger)  
- My Functions Server (MCP Trigger)  
- Google Calendar Tool Nodes: SearchEvent, CreateEvent, UpdateEvent, DeleteEvent  

**Node Details:**

- **Google Calendar MCP**  
  - Type: `@n8n/n8n-nodes-langchain.mcpTrigger`  
  - Role: MCP Server exposing Google Calendar tools.  
  - Config: Webhook path `my-calendar`.  
  - Inputs: Receives AI Agent tool calls.  
  - Outputs: To Google Calendar tool nodes.  
  - Version: Requires n8n 1.88.0+.  
  - Edge cases: Webhook activation required, URL must be copied to MCP Client.

- **My Functions Server**  
  - Type: `@n8n/n8n-nodes-langchain.mcpTrigger`  
  - Role: MCP Server exposing custom function tools.  
  - Config: Webhook path `my-functions`.  
  - Inputs: Receives AI Agent tool calls.  
  - Outputs: To sub-workflows for custom functions.  
  - Version: Requires n8n 1.88.0+.  
  - Edge cases: Same as above.

- **Google Calendar Tool Nodes**  
  - Types: `n8n-nodes-base.googleCalendarTool`  
  - Role: Perform calendar operations (search, create, update, delete).  
  - Config: Use OAuth2 credentials for Google Calendar (`Google Calendar account`).  
  - Inputs: From MCP Trigger node.  
  - Outputs: Back to MCP Trigger.  
  - Key expressions: Use `$fromAI()` to dynamically receive parameters like event times, IDs, titles, descriptions.  
  - Edge cases: Auth errors, invalid event IDs, API limits, date format errors.

---

#### 2.3 MCP Clients Setup

**Overview:**  
Connects the AI Agent to the MCP Servers via MCP Client nodes, enabling dynamic tool usage.

**Nodes Involved:**  
- Calendar MCP (MCP Client Tool)  
- My Functions (MCP Client Tool)  

**Node Details:**

- **Calendar MCP**  
  - Type: `@n8n/n8n-nodes-langchain.mcpClientTool`  
  - Role: Connects AI Agent to Google Calendar MCP Server.  
  - Config: SSE endpoint URL `https://n8n.yourdomain/mcp/my-calendar/sse` (placeholder to be replaced).  
  - Inputs: From AI Agent.  
  - Outputs: To AI Agent.  
  - Edge cases: SSE connection failures, URL misconfiguration.

- **My Functions**  
  - Type: `@n8n/n8n-nodes-langchain.mcpClientTool`  
  - Role: Connects AI Agent to Custom Functions MCP Server.  
  - Config: SSE endpoint URL `https://n8n.yourdomain/mcp/my-functions/sse` (placeholder).  
  - Inputs: From AI Agent.  
  - Outputs: To AI Agent.  
  - Edge cases: Same as above.

---

#### 2.4 Custom Function Sub-Workflows

**Overview:**  
Implements custom tools as sub-workflows callable by the MCP Server for text conversion, random user data generation, and joke retrieval.

**Nodes Involved:**  
- Convert Text (Tool Workflow)  
- Generate random user data (Tool Workflow)  
- Random Jokes (Tool Workflow)  
- When Executed by Another Workflow (Execute Workflow Trigger)  
- Switch  
- Convert Text to Upper Case (Set)  
- Convert Text to Lower Case (Set)  
- Random user data (Debug Helper)  
- Return only some fields (Set)  
- Joke Request (HTTP Request)  

**Node Details:**

- **Convert Text**  
  - Type: `@n8n/n8n-nodes-langchain.toolWorkflow`  
  - Role: Tool to convert text case (upper/lower).  
  - Config: Calls sub-workflow within same workflow by ID.  
  - Inputs: `function_name` ("uppercase" or "lowercase") and `payload` with text.  
  - Outputs: Converted text.  
  - Edge cases: Invalid function_name, missing text.

- **Generate random user data**  
  - Type: `@n8n/n8n-nodes-langchain.toolWorkflow`  
  - Role: Generates random user data entries.  
  - Config: Calls sub-workflow with parameter `number` of users.  
  - Outputs: User data array.  
  - Edge cases: Invalid number, API failures.

- **Random Jokes**  
  - Type: `@n8n/n8n-nodes-langchain.toolWorkflow`  
  - Role: Obtains random jokes from external API.  
  - Config: Calls sub-workflow with parameter `number` of jokes.  
  - Outputs: Joke data.  
  - Edge cases: API downtime, invalid number.

- **When Executed by Another Workflow**  
  - Type: `n8n-nodes-base.executeWorkflowTrigger`  
  - Role: Entry point for sub-workflows called by MCP Server.  
  - Inputs: `function_name` and `payload` parameters.  
  - Outputs: To Switch node.  
  - Edge cases: Missing parameters.

- **Switch**  
  - Type: `n8n-nodes-base.switch`  
  - Role: Routes execution based on `function_name` to appropriate processing nodes.  
  - Outputs: Uppercase, Lowercase, Random Data, Joke branches.  
  - Edge cases: Unknown function_name values.

- **Convert Text to Upper Case / Lower Case**  
  - Type: `n8n-nodes-base.set`  
  - Role: Performs string case conversion using expressions.  
  - Inputs: From Switch node.  
  - Outputs: To MCP Server response.  
  - Edge cases: Non-string inputs.

- **Random user data (Debug Helper)**  
  - Type: `n8n-nodes-base.debugHelper`  
  - Role: Generates random data count based on input.  
  - Outputs: To Set node filtering fields.  
  - Edge cases: Invalid count.

- **Return only some fields**  
  - Type: `n8n-nodes-base.set`  
  - Role: Filters random user data to only first name, last name, and email.  
  - Inputs: From Debug Helper.  
  - Outputs: To MCP Server response.  
  - Edge cases: Missing fields in data.

- **Joke Request**  
  - Type: `n8n-nodes-base.httpRequest`  
  - Role: Calls external joke API with requested number of jokes.  
  - Inputs: From Switch node.  
  - Outputs: To MCP Server response.  
  - Edge cases: API errors, invalid URL parameters.

---

#### 2.5 Workflow Control and Routing

**Overview:**  
Manages routing of function calls from MCP Server to appropriate sub-workflows or nodes.

**Nodes Involved:**  
- Switch  
- When Executed by Another Workflow  

**Node Details:**

- **Switch**  
  - Routes based on `function_name` to the correct processing branch (uppercase, lowercase, random user data, joke).  
  - Ensures modular handling of different custom functions.

- **When Executed by Another Workflow**  
  - Acts as a trigger for sub-workflows invoked by MCP Server tools.  
  - Receives parameters and passes to Switch for routing.

---

#### 2.6 Utility and Debugging

**Overview:**  
Includes nodes for debugging, instructional sticky notes, and workflow metadata.

**Nodes Involved:**  
- Sticky Note (multiple)  
- Debug Helper (Random user data)  

**Node Details:**

- **Sticky Notes**  
  - Provide instructions on activating workflow, setting credentials, example requests, author info, and help links.  
  - Positioned near relevant nodes for user guidance.

- **Debug Helper**  
  - Used to generate random data for testing and demonstration.

---

### 3. Summary Table

| Node Name                  | Node Type                               | Functional Role                          | Input Node(s)                 | Output Node(s)                 | Sticky Note                                                                                         |
|----------------------------|---------------------------------------|----------------------------------------|------------------------------|-------------------------------|---------------------------------------------------------------------------------------------------|
| When chat message received | @n8n/n8n-nodes-langchain.chatTrigger | Entry point for chat messages           | External webhook              | Simple Memory                  |                                                                                                   |
| Simple Memory              | @n8n/n8n-nodes-langchain.memoryBufferWindow | Maintains AI conversation context      | When chat message received    | OpenAI 4o                     |                                                                                                   |
| OpenAI 4o                 | @n8n/n8n-nodes-langchain.lmChatOpenAi | Language model for AI responses         | Simple Memory                 | AI Agent                      | Sticky Note7: Why model 4o? Explains model choice for calendar handling.                           |
| AI Agent                  | @n8n/n8n-nodes-langchain.agent         | Central AI orchestrator                  | OpenAI 4o, MCP Clients        | MCP Clients, Simple Memory    | Sticky Note: Activate workflow to enable MCP Trigger; copy Production URL to MCP Client.          |
| Google Calendar MCP       | @n8n/n8n-nodes-langchain.mcpTrigger    | MCP Server exposing Google Calendar tools | AI Agent                     | Google Calendar Tool Nodes    | Sticky Note4: Google Calendar tools require credentials; tutorial video link provided.            |
| My Functions Server       | @n8n/n8n-nodes-langchain.mcpTrigger    | MCP Server exposing custom function tools | AI Agent                     | Sub-workflows (Convert Text, Random Jokes, etc.) | Sticky Note1: MCP Clients need Production URLs from MCP Triggers.                                |
| SearchEvent               | n8n-nodes-base.googleCalendarTool      | Search calendar events                   | Google Calendar MCP           | Google Calendar MCP           |                                                                                                   |
| CreateEvent               | n8n-nodes-base.googleCalendarTool      | Create calendar event                    | Google Calendar MCP           | Google Calendar MCP           |                                                                                                   |
| UpdateEvent               | n8n-nodes-base.googleCalendarTool      | Update calendar event                    | Google Calendar MCP           | Google Calendar MCP           |                                                                                                   |
| DeleteEvent               | n8n-nodes-base.googleCalendarTool      | Delete calendar event                    | Google Calendar MCP           | Google Calendar MCP           |                                                                                                   |
| Calendar MCP              | @n8n/n8n-nodes-langchain.mcpClientTool | Connects AI Agent to Google Calendar MCP Server | AI Agent                     | AI Agent                     | Sticky Note1: MCP Clients need Production URLs from MCP Triggers.                                |
| My Functions              | @n8n/n8n-nodes-langchain.mcpClientTool | Connects AI Agent to Custom Functions MCP Server | AI Agent                     | AI Agent                     | Sticky Note1: MCP Clients need Production URLs from MCP Triggers.                                |
| Convert Text              | @n8n/n8n-nodes-langchain.toolWorkflow  | Tool for text case conversion            | My Functions Server           | My Functions Server           | Sticky Note3: Explains sub-workflow usage and link to docs.                                      |
| Generate random user data | @n8n/n8n-nodes-langchain.toolWorkflow  | Tool for generating random user data    | My Functions Server           | My Functions Server           | Sticky Note3: Explains sub-workflow usage and link to docs.                                      |
| Random Jokes              | @n8n/n8n-nodes-langchain.toolWorkflow  | Tool for obtaining random jokes          | My Functions Server           | My Functions Server           | Sticky Note3: Explains sub-workflow usage and link to docs.                                      |
| When Executed by Another Workflow | n8n-nodes-base.executeWorkflowTrigger | Entry point for sub-workflow calls       | My Functions Server           | Switch                       |                                                                                                   |
| Switch                    | n8n-nodes-base.switch                   | Routes function calls to appropriate nodes | When Executed by Another Workflow | Convert Text to Upper Case, Convert Text to Lower Case, Random user data, Joke Request |                                                                                                   |
| Convert Text to Upper Case | n8n-nodes-base.set                     | Converts text to uppercase                | Switch (UPPERCASE)            | My Functions Server           |                                                                                                   |
| Convert Text to Lower Case | n8n-nodes-base.set                     | Converts text to lowercase                | Switch (LOWERCASE)            | My Functions Server           |                                                                                                   |
| Random user data          | n8n-nodes-base.debugHelper             | Generates random user data for testing   | Switch (RANDOM DATA)          | Return only some fields       |                                                                                                   |
| Return only some fields   | n8n-nodes-base.set                     | Filters random user data fields           | Random user data              | My Functions Server           |                                                                                                   |
| Joke Request              | n8n-nodes-base.httpRequest             | Calls external API to get jokes           | Switch (JOKE)                 | My Functions Server           |                                                                                                   |
| Sticky Note (multiple)    | n8n-nodes-base.stickyNote              | Instructions, author info, help links     | N/A                          | N/A                          | Various notes with instructions, links to tutorials, author info, and community help.             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the AI Agent Setup:**

   - Add **When chat message received** node (chatTrigger) with default webhook.
   - Add **Simple Memory** node (memoryBufferWindow) connected from chat trigger.
   - Add **OpenAI 4o** node (lmChatOpenAi):
     - Model: Select `gpt-4o`.
     - Credentials: Configure OpenAI API credentials.
     - Connect from Simple Memory.
   - Add **AI Agent** node (agent):
     - System message: `"You are a helpful assistant.\nCurrent datetime is {{ $now.toString() }}"`.
     - Connect from OpenAI 4o node.
     - Connect AI Agent outputs to MCP Clients (to be created).

2. **Set up Google Calendar MCP Server:**

   - Add **Google Calendar MCP** node (mcpTrigger):
     - Path: `my-calendar`.
     - Activate workflow to generate Production URL.
   - Add Google Calendar tool nodes connected from MCP Trigger:
     - **SearchEvent** (googleCalendarTool):
       - Operation: `getAll`.
       - Calendar: Select your Google Calendar account.
       - Parameters: Use expressions to accept inputs from AI (`$fromAI()`).
       - Credentials: Google Calendar OAuth2.
     - **CreateEvent** (googleCalendarTool):
       - Operation: `create`.
       - Calendar: Same as above.
       - Additional fields: summary, description from AI inputs.
       - Credentials: Same.
     - **UpdateEvent** (googleCalendarTool):
       - Operation: `update`.
       - Event ID and fields from AI inputs.
       - Credentials: Same.
     - **DeleteEvent** (googleCalendarTool):
       - Operation: `delete`.
       - Event ID from AI inputs.
       - Credentials: Same.
   - Connect all Google Calendar tool nodes back to MCP Trigger node.

3. **Set up Custom Functions MCP Server:**

   - Add **My Functions Server** node (mcpTrigger):
     - Path: `my-functions`.
     - Activate workflow to generate Production URL.
   - Add sub-workflows for custom functions (see next step).
   - Connect MCP Trigger to sub-workflow execution trigger node.

4. **Create Sub-Workflows for Custom Functions:**

   - Create a new workflow or use the same workflow with sub-workflow nodes.
   - Add **When Executed by Another Workflow** node:
     - Inputs: `function_name` (string), `payload` (object).
   - Add **Switch** node connected from above:
     - Routes based on `function_name` values: `uppercase`, `lowercase`, `random_user_data`, `joke`.
   - For `uppercase` branch:
     - Add **Set** node to convert `payload.text` to uppercase.
   - For `lowercase` branch:
     - Add **Set** node to convert `payload.text` to lowercase.
   - For `random_user_data` branch:
     - Add **Debug Helper** node:
       - Category: `randomData`.
       - Count: from `payload.number`.
     - Add **Set** node to filter fields: First name, Last name, Email.
   - For `joke` branch:
     - Add **HTTP Request** node:
       - URL: `https://official-joke-api.appspot.com/jokes/random/{{ $json.payload.number }}`.
   - Connect all branches back to the MCP Server response.

5. **Set up MCP Clients:**

   - Add **Calendar MCP** node (mcpClientTool):
     - SSE Endpoint: Paste Production URL from Google Calendar MCP Trigger + `/sse`.
     - Connect AI Agent output to this node.
   - Add **My Functions** node (mcpClientTool):
     - SSE Endpoint: Paste Production URL from My Functions MCP Trigger + `/sse`.
     - Connect AI Agent output to this node.
   - Connect MCP Clients outputs back to AI Agent.

6. **Connect AI Agent inputs and outputs:**

   - From **When chat message received** → **Simple Memory** → **OpenAI 4o** → **AI Agent**.
   - From **AI Agent** → **Calendar MCP** and **My Functions** MCP Clients.
   - From MCP Clients → back to **AI Agent**.

7. **Configure Credentials:**

   - Set up Google Calendar OAuth2 credentials with appropriate scopes.
   - Set up OpenAI API credentials.
   - Ensure webhook URLs are publicly accessible or use n8n cloud.

8. **Activate the workflow.**

9. **Test with example prompts:**

   - Use example requests from Sticky Note2 for testing text conversion, random data, jokes, and calendar events.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                     | Context or Link                                                                                     |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| Activate the workflow to make the MCP Trigger work. Copy the Production URL of the MCP Trigger and paste it in the corresponding MCP Client tool.                                                                              | Sticky Note near AI Agent and MCP Trigger nodes.                                                  |
| Google Calendar tools require credentials. If you don't have your Google Credentials set up in n8n yet, watch this video: https://www.youtube.com/watch?v=3Ai1EPznlAc                                                           | Sticky Note4 near Google Calendar MCP nodes.                                                     |
| Try these example requests with the AI Agent: convert text case, generate random user data, obtain jokes, manage calendar events.                                                                                              | Sticky Note2 near AI Agent input.                                                                 |
| The My Functions MCP calls this sub-workflow every time. Learn more about sub-workflows here: https://docs.n8n.io/flow-logic/subworkflows/                                                                                      | Sticky Note3 near sub-workflow nodes.                                                             |
| Why model 4o? After testing 4o-mini, it had difficulties handling calendar requests, while 4o model handled it with ease. Depending on your prompt and tools, 4o-mini might work but requires testing.                            | Sticky Note7 near OpenAI 4o node.                                                                 |
| Author: Solomon, Freelance consultant from Brazil specializing in automations and data analysis. Check out other templates: https://n8n.io/creators/solomon/ Business inquiries: automations.solomon@gmail.com Telegram: @salomaoguilherme | Sticky Note5 near workflow metadata.                                                              |
| Need help? For support, create a topic on the community forums: https://community.n8n.io/c/questions/                                                                                                                           | Sticky Note6 near bottom of workflow.                                                             |
| Learn How to Build an MCP Server and Client                                                                                                                                                                                     | Sticky Note9 at top of workflow.                                                                  |

---

This documentation provides a complete understanding of the workflow structure, node configurations, and instructions to reproduce or modify the MCP Server and Client setup with Google Calendar and custom functions in n8n. It also highlights potential failure points and integration considerations.