Starred Slack Messages to Notion Database with AI Auto-Tagging

https://n8nworkflows.xyz/workflows/starred-slack-messages-to-notion-database-with-ai-auto-tagging-4502


# Starred Slack Messages to Notion Database with AI Auto-Tagging

---

### 1. Workflow Overview

This workflow automates the process of capturing starred messages from Slack, enriching them with AI-generated tags and titles, and then saving them into a Notion database. The primary use case is to help users organize important Slack messages by automatically tagging and storing them in a structured Notion page for easy reference.

The workflow is logically divided into the following blocks:

- **1.1 Scheduled Slack Message Retrieval:** Periodically fetches Slack messages.
- **1.2 Star Reaction Filter:** Filters messages that have been starred.
- **1.3 Message Link Retrieval:** Gets the permalink for the starred Slack messages.
- **1.4 Notion Database Selection:** Chooses the target Notion database to store data.
- **1.5 AI Processing for Title and Tagging:** Uses OpenAI's language model to generate relevant titles and tags for the message content.
- **1.6 Notion Page Creation:** Creates a new page in Notion with the enriched data.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Scheduled Slack Message Retrieval

- **Overview:** This block initiates the workflow on a scheduled basis and retrieves recent Slack messages for processing.
  
- **Nodes Involved:**
  - Schedule Trigger
  - Get Slack Messages

- **Node Details:**

  - **Schedule Trigger**
    - Type: Trigger node for scheduled execution.
    - Configuration: Default scheduling parameters (not explicitly specified).
    - Input: None (starting node).
    - Output: Triggers "Get Slack Messages".
    - Failure Modes: Scheduling misconfiguration, node downtime.

  - **Get Slack Messages**
    - Type: Slack node (API integration).
    - Configuration: Fetches messages from Slack channels; uses configured Slack credentials.
    - Input: Trigger from Schedule Trigger.
    - Output: Sends messages to "IF reaction == star".
    - Failure Modes: Slack API rate limits, authentication errors, network issues.

---

#### 2.2 Star Reaction Filter

- **Overview:** This conditional block filters only those Slack messages which have been starred by users.

- **Nodes Involved:**
  - IF reaction == star

- **Node Details:**

  - **IF reaction == star**
    - Type: Conditional (If) node.
    - Configuration: Checks if the reaction on a Slack message equals "star".
    - Input: Slack messages from "Get Slack Messages".
    - Output: If true, passes messages to "Get Message Link".
    - Failure Modes: Missing or malformed reaction data, expression errors.

---

#### 2.3 Message Link Retrieval

- **Overview:** Retrieves the direct permalink URL of the starred Slack messages for reference.

- **Nodes Involved:**
  - Get Message Link

- **Node Details:**

  - **Get Message Link**
    - Type: Slack node.
    - Configuration: Calls Slack API to retrieve permalink for a specific message.
    - Input: Filtered messages from "IF reaction == star".
    - Output: Passes enriched messages with links to "Choose Notion DB".
    - Failure Modes: Slack API errors, message not found, permission issues.

---

#### 2.4 Notion Database Selection

- **Overview:** Selects the appropriate Notion database where the message data will be stored.

- **Nodes Involved:**
  - Choose Notion DB

- **Node Details:**

  - **Choose Notion DB**
    - Type: Notion node.
    - Configuration: Specifies the target Notion database.
    - Input: Messages with links from "Get Message Link".
    - Output: Passes data to "Set Tags".
    - Failure Modes: Invalid database ID, authentication errors.

---

#### 2.5 AI Processing for Title and Tagging

- **Overview:** Uses OpenAI's language model to generate structured tags and titles for the Slack messages to improve organization.

- **Nodes Involved:**
  - Set Tags
  - OpenAI Chat Model
  - Structured Output Parser
  - Write Title & Tag

- **Node Details:**

  - **Set Tags**
    - Type: Set node.
    - Configuration: Prepares or initializes data fields for AI processing.
    - Input: Data from "Choose Notion DB".
    - Output: Sends to "Write Title & Tag".
    - Failure Modes: Misconfigured fields or variables.

  - **OpenAI Chat Model**
    - Type: Langchain OpenAI Chat Model node.
    - Configuration: Uses OpenAI API to generate AI responses.
    - Input: Triggered via AI language model connection from "Write Title & Tag".
    - Output: AI-generated content passed to "Structured Output Parser".
    - Failure Modes: API key issues, rate limits, prompt errors.

  - **Structured Output Parser**
    - Type: Langchain output parser.
    - Configuration: Parses AI output into a structured format.
    - Input: AI responses from "OpenAI Chat Model".
    - Output: Structured data sent back to "Write Title & Tag".
    - Failure Modes: Parsing errors, unexpected AI response format.

  - **Write Title & Tag**
    - Type: Langchain agent node.
    - Configuration: Combines AI-generated title and tags, formats data for Notion.
    - Input: From "Set Tags" and parsed AI output.
    - Output: Sends final data to "Create Notion Page".
    - Failure Modes: Expression errors, data formatting issues.

---

#### 2.6 Notion Page Creation

- **Overview:** Creates a new page in the selected Notion database with the Slack message, AI-generated title, tags, and link.

- **Nodes Involved:**
  - Create Notion Page

- **Node Details:**

  - **Create Notion Page**
    - Type: Notion node.
    - Configuration: Uses Notion API to create a page with properties (message content, title, tags, link).
    - Input: Final structured data from "Write Title & Tag".
    - Output: End of workflow.
    - Failure Modes: API permission errors, invalid property fields, rate limiting.

---

### 3. Summary Table

| Node Name           | Node Type                         | Functional Role                 | Input Node(s)         | Output Node(s)        | Sticky Note                        |
|---------------------|----------------------------------|--------------------------------|-----------------------|-----------------------|----------------------------------|
| Schedule Trigger     | Schedule Trigger                 | Initiates workflow periodically| None                  | Get Slack Messages     |                                  |
| Get Slack Messages   | Slack                           | Fetches Slack messages         | Schedule Trigger       | IF reaction == star    |                                  |
| IF reaction == star  | If                             | Filters starred messages       | Get Slack Messages     | Get Message Link       |                                  |
| Get Message Link     | Slack                           | Retrieves message permalink    | IF reaction == star    | Choose Notion DB       |                                  |
| Choose Notion DB     | Notion                          | Selects target Notion database | Get Message Link       | Set Tags               |                                  |
| Set Tags             | Set                            | Initializes data for AI        | Choose Notion DB       | Write Title & Tag      |                                  |
| OpenAI Chat Model    | Langchain OpenAI Chat Model     | Generates AI-based tags/titles| Write Title & Tag (AI) | Structured Output Parser|                                  |
| Structured Output Parser | Langchain Output Parser        | Parses AI output               | OpenAI Chat Model      | Write Title & Tag      |                                  |
| Write Title & Tag    | Langchain Agent                 | Formats AI data for Notion     | Set Tags, Structured Output Parser | Create Notion Page |                                  |
| Create Notion Page   | Notion                          | Creates Notion page            | Write Title & Tag      | None                  |                                  |
| Sticky Note          | Sticky Note                    | Comments/Notes                 | None                  | None                  | None (empty note)                |
| Sticky Note1         | Sticky Note                    | Comments/Notes                 | None                  | None                  | None (empty note)                |
| Sticky Note2         | Sticky Note                    | Comments/Notes                 | None                  | None                  | None (empty note)                |
| Sticky Note3         | Sticky Note                    | Comments/Notes                 | None                  | None                  | None (empty note)                |
| Sticky Note4         | Sticky Note                    | Comments/Notes                 | None                  | None                  | None (empty note)                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**
   - Set to trigger at desired intervals (e.g., every 5 minutes).
   - No input.
   - Output connected to "Get Slack Messages".

2. **Create a Slack node named "Get Slack Messages"**
   - Configure Slack OAuth2 credentials.
   - Set API call to fetch recent messages from channels.
   - Connect input from Schedule Trigger.
   - Output connected to "IF reaction == star".

3. **Create an If node named "IF reaction == star"**
   - Condition: Check if the message's reactions contain a "star".
   - Input from "Get Slack Messages".
   - True output connected to "Get Message Link".
   - False output left unconnected or handled as needed.

4. **Create a Slack node named "Get Message Link"**
   - Configure Slack credentials.
   - Use Slack API method to get permalink of the message.
   - Input from "IF reaction == star" true branch.
   - Output connected to "Choose Notion DB".

5. **Create a Notion node named "Choose Notion DB"**
   - Configure Notion credentials.
   - Set to the target Notion database ID where pages will be created.
   - Input from "Get Message Link".
   - Output connected to "Set Tags".

6. **Create a Set node named "Set Tags"**
   - Prepare or initialize fields needed for AI processing (e.g., message text).
   - Input from "Choose Notion DB".
   - Output connected to "Write Title & Tag".

7. **Create a Langchain OpenAI Chat Model node named "OpenAI Chat Model"**
   - Configure OpenAI API credentials.
   - Set model (e.g., GPT-4 or GPT-3.5) and parameters for tagging and titling.
   - No direct input; this node is linked via AI language model connection.
   - Output connected to "Structured Output Parser".

8. **Create a Langchain Output Parser node named "Structured Output Parser"**
   - Configure to parse AI output into structured JSON with title and tags.
   - Input from "OpenAI Chat Model".
   - Output connected to "Write Title & Tag" via AI output parser connection.

9. **Create a Langchain Agent node named "Write Title & Tag"**
   - Combines AI-generated title and tags with Slack message data.
   - Input from "Set Tags" (main) and AI output parser (ai_outputParser).
   - AI language model input connected to "OpenAI Chat Model".
   - Output connected to "Create Notion Page".

10. **Create a Notion node named "Create Notion Page"**
    - Configure Notion credentials.
    - Set action to create a new page in the chosen database.
    - Map properties: message content, AI-generated title, tags, and Slack permalink.
    - Input from "Write Title & Tag".
    - No output (workflow end).

11. **Optional: Add Sticky Note nodes as needed to document or comment on workflow sections.**

---

### 5. General Notes & Resources

| Note Content                                                                                       | Context or Link                         |
|--------------------------------------------------------------------------------------------------|---------------------------------------|
| This workflow requires valid OAuth2 credentials for both Slack and Notion APIs.                  | Credential setup in n8n                |
| OpenAI API key with access to chat models (GPT-4 or GPT-3.5) is needed for AI processing.        | https://platform.openai.com/account/api-keys |
| Ensure Slack app has permissions to read message reactions and retrieve permalinks.              | Slack API scopes: reactions:read, chat:read, links:read |
| Notion integration must have access to the target database and rights to create pages.           | Notion API documentation: https://developers.notion.com/ |
| Rate limiting on Slack and OpenAI APIs might affect execution; consider implementing error handling and retries. | See API docs for limits and best practices |
| AI parsing depends on consistent response formatting; monitor AI output changes for robustness.  | Use Langchain structured parsers       |

---

**Disclaimer:** The provided content originates exclusively from an automated workflow created with n8n, a workflow automation tool. This processing strictly complies with applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.

---