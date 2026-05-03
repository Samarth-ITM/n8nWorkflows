Automatically Triage & Improve Todoist Tasks with GPT-4.1-mini

https://n8nworkflows.xyz/workflows/automatically-triage---improve-todoist-tasks-with-gpt-4-1-mini-7938


# Automatically Triage & Improve Todoist Tasks with GPT-4.1-mini

### 1. Workflow Overview

This workflow, titled **"Automatically Triage & Improve Todoist Tasks with GPT-4.1-mini"**, is designed to intelligently manage new tasks added to a Todoist account by leveraging AI (an n8n Langchain AI Agent powered by OpenAI GPT-4.1-mini). Its primary use case is to automatically triage, prioritize, and improve task descriptions based on the context of existing open tasks, ensuring realistic scheduling and enhanced task clarity.

The workflow logic can be grouped into four main blocks:

- **1.1 Input Reception & Trigger**: Periodically detect newly created tasks in Todoist.
- **1.2 Task Context Acquisition**: Retrieve all current open tasks to provide backlog context.
- **1.3 AI Processing & Decision Making**: Use an AI agent to analyze the new task in the context of open tasks, decide priority, improve description, and schedule a due date.
- **1.4 Task Update Execution**: Update the original task in Todoist with the AI-generated improvements.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Trigger

- **Overview:**  
  This block periodically triggers the workflow and fetches tasks created within the last 5 minutes from Todoist.

- **Nodes Involved:**  
  - Schedule Trigger  
  - Get many tasks (Todoist)

- **Node Details:**

  - **Schedule Trigger**  
    - *Type & Role:* Native n8n node to trigger workflows on a schedule.  
    - *Configuration:* Set to run at intervals defined by minutes (default interval unspecified, but triggers frequently).  
    - *Inputs:* None (trigger node).  
    - *Outputs:* Triggers "Get many tasks".  
    - *Edge Cases:* Potential delays if n8n is down; no direct failure modes beyond n8n runtime errors.

  - **Get many tasks (Todoist)**  
    - *Type & Role:* Todoist node fetching tasks.  
    - *Configuration:* Retrieves all tasks created after "now - 5 minutes" to capture new tasks.  
    - *Credentials:* Uses stored Todoist API credentials.  
    - *Inputs:* Triggered by Schedule Trigger.  
    - *Outputs:* Sends new tasks data to AI Agent.  
    - *Edge Cases:* API rate limits, connectivity issues, or empty task sets if no new tasks found.

---

#### 2.2 Task Context Acquisition

- **Overview:**  
  Retrieve the current backlog of open tasks from Todoist to provide the AI agent context for prioritization and scheduling decisions.

- **Nodes Involved:**  
  - get_open_tasks (Todoist Tool)

- **Node Details:**

  - **get_open_tasks**  
    - *Type & Role:* Todoist tool node to fetch all open tasks.  
    - *Configuration:* Retrieves all open tasks without filters, ignoring completed or archived tasks.  
    - *Credentials:* Uses Todoist API credentials.  
    - *Inputs:* Connected as an AI tool input for the AI Agent node.  
    - *Outputs:* Provides full backlog data for AI processing.  
    - *Edge Cases:* API failures or empty backlog; recurring tasks and "inbox cleanup" tasks are handled logically in AI prompt instructions.

---

#### 2.3 AI Processing & Decision Making

- **Overview:**  
  The heart of the workflow where the AI agent receives the new task and backlog context, applies a structured framework to decide task priority, improves the task description, and schedules a realistic due date.

- **Nodes Involved:**  
  - AI Agent (Langchain Agent)  
  - OpenAI Chat Model (GPT-4.1-mini)

- **Node Details:**

  - **AI Agent**  
    - *Type & Role:* Langchain agent node that coordinates AI tools and manages prompt logic.  
    - *Configuration:*  
      - Uses a detailed system message defining the agent’s identity, mission, golden rules, and instructions for priority calculation and scheduling.  
      - Task input is templated with expressions injecting dynamic task data: id, content, labels, due_hint, project_id, timebox_min.  
      - Defines two tools:  
        1) `get_open_tasks` to read current backlog,  
        2) `update_task` to apply improvements.  
      - Uses strict scheduling policies and a priority scoring framework with clear numeric thresholds.  
      - Provides directives for rewriting task description into clear imperative style with completion criteria.  
      - Enforces rules like never leaving a task without a due date, respecting recurring tasks, weekend offs, and urgent tagging.  
    - *Inputs:* Receives new tasks from "Get many tasks" node, and backlog from "get_open_tasks" tool node.  
    - *Outputs:* Sends AI-generated updates (task ID, improved description, priority, due date) to "update_task".  
    - *Edge Cases:*  
      - AI response failures or timeouts.  
      - Misinterpretation of priorities or scheduling rules (mitigated by strict prompt).  
      - Unexpected null or malformed input data.  
    - *Version Requirements:* Uses Langchain agent v2.2 features, including AI tool chaining and prompt templating.

  - **OpenAI Chat Model (GPT-4.1-mini)**  
    - *Type & Role:* Language model node providing GPT-4.1-mini chat completions.  
    - *Configuration:* Model set to "gpt-4.1-mini" with default options.  
    - *Credentials:* Uses OpenAI API credentials.  
    - *Inputs:* Receives prompts from AI Agent node.  
    - *Outputs:* Returns AI-generated text for task triage and improvement.  
    - *Edge Cases:*  
      - API rate limiting or connectivity errors.  
      - Model output variability or hallucinations.  
      - Ensuring compliance with token limits.  
    - *Version Requirements:* Requires n8n version supporting Langchain nodes with OpenAI integration.

---

#### 2.4 Task Update Execution

- **Overview:**  
  Apply the AI agent's decisions by updating the original task in Todoist with improved description, assigned priority, and scheduled due date.

- **Nodes Involved:**  
  - update_task (Todoist Tool)

- **Node Details:**

  - **update_task**  
    - *Type & Role:* Todoist tool node to update a specific task.  
    - *Configuration:*  
      - Updates task fields: priority, description, dueDateTime using AI agent outputs.  
      - Task ID is dynamically set from AI output.  
      - Uses expressions referencing AI agent's structured output variables such as Task_ID, Priority, Description, Due_Date_Time.  
    - *Credentials:* Uses Todoist API credentials.  
    - *Inputs:* Receives AI agent’s output.  
    - *Outputs:* None (terminal node).  
    - *Edge Cases:*  
      - Invalid task ID or permissions errors.  
      - API update failures or conflicts.  
      - Format errors in due date/time string (must be RFC3339 UTC).  
    - *Version Requirements:* Supports Todoist API v2.1 features.

---

### 3. Summary Table

| Node Name         | Node Type                      | Functional Role              | Input Node(s)          | Output Node(s)        | Sticky Note                                                                                                    |
|-------------------|--------------------------------|-----------------------------|-----------------------|-----------------------|---------------------------------------------------------------------------------------------------------------|
| Schedule Trigger  | n8n-nodes-base.scheduleTrigger | Periodic workflow trigger   | None                  | Get many tasks        | ### Task Triage Agent Automatically adapts new tasks taking in account other tasks the user already has       |
| Get many tasks    | n8n-nodes-base.todoist         | Fetch new tasks created last 5 minutes | Schedule Trigger      | AI Agent              | - Configure credentials in the **TodoIst** and **Model** nodes                                               |
| get_open_tasks    | n8n-nodes-base.todoistTool     | Retrieve all open tasks (backlog) | AI Agent (ai_tool)    | AI Agent              | - Click **Execute workflow** button to test with the last email in your inbox                                  |
| AI Agent          | @n8n/n8n-nodes-langchain.agent | AI logic for triage, priority, scheduling | Get many tasks, get_open_tasks | update_task           |                                                                                                               |
| OpenAI Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | Language model providing GPT-4.1-mini completions | AI Agent (ai_languageModel) | AI Agent              |                                                                                                               |
| update_task       | n8n-nodes-base.todoistTool     | Update Todoist task with AI improvements | AI Agent (ai_tool)    | None                  |                                                                                                               |
| Sticky Note       | n8n-nodes-base.stickyNote      | Informational setup instructions | None                  | None                  | ### Task Triage Agent Automatically adapts new tasks taking in account other tasks the user already has       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a "Schedule Trigger" node:**  
   - Node Type: `n8n-nodes-base.scheduleTrigger`  
   - Set to trigger periodically by minutes (default interval or custom, e.g., every 5 minutes).  
   - No credentials needed.

3. **Add a "Get many tasks" Todoist node:**  
   - Node Type: `n8n-nodes-base.todoist`  
   - Operation: `getAll`  
   - Filters: Set filter to `"created after: -5 minutes"` to fetch newly created tasks.  
   - Connect Schedule Trigger main output to this node's input.  
   - Set Todoist API credentials (OAuth2 or token).  
   - Enable `Return All`.

4. **Add a "get_open_tasks" Todoist Tool node:**  
   - Node Type: `n8n-nodes-base.todoistTool`  
   - Operation: `getAll` with no filters to fetch all open tasks.  
   - Set Todoist API credentials (same as above).  
   - This node will be used as a tool input for the AI Agent.

5. **Add an "OpenAI Chat Model" node:**  
   - Node Type: `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
   - Model: Select `"gpt-4.1-mini"`.  
   - Set OpenAI API credentials (API key).  
   - No additional options required.

6. **Add an "AI Agent" Langchain Agent node:**  
   - Node Type: `@n8n/n8n-nodes-langchain.agent`  
   - Configure the `text` parameter with the provided prompt template, injecting new task details using expressions like `{{ $json.id }}`, `{{ $json.content }}`, etc.  
   - Configure system message with the detailed instructions for task triage, priority scoring, scheduling rules, description improvement, and use of tools `get_open_tasks` and `update_task`.  
   - Set `get_open_tasks` and `update_task` as AI tools available to the agent.  
   - Connect inputs:  
     - Main input connected from "Get many tasks" (new tasks).  
     - AI Tool input connected with "get_open_tasks".  
     - AI Language Model input connected with "OpenAI Chat Model".  
   - Enable version 2.2 features as applicable.

7. **Add an "update_task" Todoist Tool node:**  
   - Node Type: `n8n-nodes-base.todoistTool`  
   - Operation: `update`  
   - Parameters:  
     - `taskId`: dynamic from AI agent output `Task_ID`.  
     - Update fields:  
       - `priority`: from AI output `Priority` (1–4).  
       - `description`: from AI output `Description`.  
       - `dueDateTime`: from AI output `Due_Date_Time` (RFC3339 UTC string).  
   - Set Todoist API credentials.  
   - Connect AI Agent's AI Tool output to this node's input.

8. **Optional: Add a Sticky Note node** for user instructions and setup info.

9. **Activate and test the workflow:**  
   - Ensure credentials are valid.  
   - Trigger manually or wait for scheduled execution.  
   - Monitor logs and execution for errors or edge cases.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                  | Context or Link                                                   |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------|
| The workflow uses a strict priority scoring framework based on Impact, Urgency, and Risk to assign Todoist priority levels efficiently.                                       | Embedded in AI Agent system prompt.                              |
| The AI agent improves task descriptions by rewriting them in imperative style and adding objective completion criteria, preserving labels, links, and references.            | Embedded in AI Agent system prompt.                              |
| Scheduling respects weekend offs and avoids overbooking by analyzing backlog task density and task duration heuristics.                                                     | Embedded in AI Agent system prompt.                              |
| Todoist priority scale clarification: priority 4 = highest (P1), down to 1 = lowest (P4).                                                                                      | Embedded in AI Agent system prompt.                              |
| Useful links for Todoist API docs and n8n Langchain integration can be found on the respective official websites for credential setup and advanced configuration.              | n8n official docs: https://docs.n8n.io/                          |
| This workflow requires proper credential setup for Todoist and OpenAI API integrations before use.                                                                             | Mentioned in sticky note and nodes.                              |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.