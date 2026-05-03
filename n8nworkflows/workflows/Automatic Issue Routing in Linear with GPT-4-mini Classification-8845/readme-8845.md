Automatic Issue Routing in Linear with GPT-4-mini Classification

https://n8nworkflows.xyz/workflows/automatic-issue-routing-in-linear-with-gpt-4-mini-classification-8845


# Automatic Issue Routing in Linear with GPT-4-mini Classification

### 1. Workflow Overview

This n8n workflow, titled **"Automatic Issue Routing in Linear with GPT-4-mini Classification"**, automates the triage and assignment of issues created or updated in the Linear issue tracking system. Its primary purpose is to classify new or updated bug reports into designated teams (Engineering, Product, Design, or a Default fallback) using an AI-powered classifier based on OpenAI's GPT-4-mini, and then automatically update the issue's team assignment in Linear.

The workflow is logically divided into four key blocks:

- **1.1 Trigger & Filtering:** Detects new or updated issues in Linear and filters out irrelevant or incomplete records.
- **1.2 AI Classification:** Uses GPT-4-mini through a Langchain AI Agent to classify issues into one of four team categories based on the issue title and description.
- **1.3 Routing Logic:** Routes the AI classification output to the appropriate path for team assignment.
- **1.4 Assignment Actions:** Updates the Linear issue with the correct team ID, effectively routing the issue to the respective team's backlog.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger & Filtering

**Overview:**  
This block listens for new or updated issues in Linear and filters them to ensure only relevant issues with titles proceed further into the workflow.

**Nodes Involved:**  
- 🔔 Linear Trigger  
- 📋 Filter New Issues Only  
- ⚖️ If (Create or Update)

**Node Details:**

- **🔔 Linear Trigger**  
  - **Type:** Linear Trigger (n8n built-in)  
  - **Role:** Entry point that subscribes to Linear issue events (create or update) for a specified team.  
  - **Configuration:**  
    - Team ID set via parameter: `"YOUR_LINEAR_TEAM_ID"` (replace with actual Linear team ID).  
    - Resource monitored: `issue`.  
    - Credentials: Linear API credentials required.  
  - **Connections:** Outputs to "📋 Filter New Issues Only".  
  - **Failure Modes:**  
    - API authentication errors if credentials invalid.  
    - Network timeouts.  
  - **Version:** 1

- **📋 Filter New Issues Only**  
  - **Type:** If (conditional node)  
  - **Role:** Ensures the issue has a non-empty title before proceeding.  
  - **Configuration:**  
    - Condition checks if `data.title` exists and is not empty.  
  - **Input:** From Linear Trigger.  
  - **Output:** True branch proceeds; False branch stops workflow.  
  - **Failure Modes:** Expression evaluation errors if input data malformed.  
  - **Version:** 2

- **⚖️ If (Create or Update)**  
  - **Type:** If (conditional node)  
  - **Role:** Filters events to only process issues that were created or updated.  
  - **Configuration:**  
    - Condition: `$json.action` equals `create` OR `update`.  
  - **Input:** From Filter New Issues Only.  
  - **Output:** True branch proceeds to AI classification.  
  - **Failure Modes:** Expression errors if `action` field missing.  
  - **Version:** 2.2

---

#### 1.2 AI Classification

**Overview:**  
This block uses an OpenAI GPT-4-mini chat model through Langchain integration and a specialized AI Agent node to classify the issue into one of four team categories based on its title and description.

**Nodes Involved:**  
- 🧠 OpenAI Chat Model  
- 🤖 AI Agent (Bug Classifier)

**Node Details:**

- **🧠 OpenAI Chat Model**  
  - **Type:** Langchain OpenAI Chat LM node  
  - **Role:** Provides the underlying language model for the AI Agent to perform classification.  
  - **Configuration:**  
    - Model selected: `gpt-4-mini`.  
    - No special options enabled beyond base.  
    - Credentials: OpenAI API credentials required.  
  - **Input:** None directly (used as resource by AI Agent).  
  - **Output:** Passed to AI Agent.  
  - **Failure Modes:**  
    - API key invalid or rate limit exceeded.  
    - Timeout or connection issues.  
  - **Version:** 1.2

- **🤖 AI Agent (Bug Classifier)**  
  - **Type:** Langchain Agent node  
  - **Role:** Classifies issues into one of Engineering, Product, Design, or Default team IDs using the LLM.  
  - **Configuration:**  
    - Prompt instructs to classify based on title and description, returning exactly one team ID string with no explanations.  
    - Uses expressions to embed:  
      - `{{ $json.data.title }}` for issue title  
      - `{{ $json.data.description }}` for issue description  
  - **Input:** From the "If (Create or Update)" node.  
  - **Output:** Classification result as team ID string (e.g., `ENGINEERING_TEAM_ID`).  
  - **Failure Modes:**  
    - Model API failures or unexpected outputs.  
    - Empty or malformed input fields causing misclassification.  
  - **Version:** 2.2

---

#### 1.3 Routing Logic

**Overview:**  
This block routes the classification output to exactly one team assignment path by matching the returned team ID.

**Nodes Involved:**  
- 🔀 Engineering Router  
- 🔀 Product Router  
- 🔀 Design Router  
- 🔀 Default Router

**Node Details:**

- **Switch Nodes (Engineering, Product, Design, Default Routers)**  
  - **Type:** Switch nodes  
  - **Role:** Each node checks if the AI Agent output matches its corresponding team ID constant. Only one Switch will match per execution.  
  - **Configuration:**  
    - Condition on `{{ $json.output }}` equals the respective team ID string.  
  - **Input:** From AI Agent node.  
  - **Output:** True branch connects to corresponding Assign node.  
  - **Failure Modes:**  
    - If AI Agent returns an unexpected or misspelled team ID, none of the routers will match, and no assignment occurs.  
  - **Version:** 3

---

#### 1.4 Assignment Actions

**Overview:**  
This block updates the Linear issue's team assignment based on the routing decision, effectively auto-routing the issue.

**Nodes Involved:**  
- 🛠️ Assign to Engineering  
- 📦 Assign to Product  
- 🎨 Assign to Design  
- 📂 Assign to Default

**Node Details:**

- **Linear Update Nodes (Assign to Engineering/Product/Design/Default)**  
  - **Type:** Linear node (update operation)  
  - **Role:** Updates the Linear issue to set its `teamId` to the appropriate team, as classified by the AI Agent.  
  - **Configuration:**  
    - `issueId` set via expression referencing the original issue ID from the "If (Create or Update)" node: `{{$('⚖️ If (Create or Update)').item.json.data.id}}`  
    - Operation: `update`  
    - `teamId` field set to the team ID string from AI Agent output.  
  - **Input:** From corresponding Router node.  
  - **Credentials:** Linear API credentials (same as trigger).  
  - **Failure Modes:**  
    - API authentication errors.  
    - Update failures if issue ID invalid or issue closed.  
  - **Version:** 1.1

---

### 3. Summary Table

| Node Name                  | Node Type                  | Functional Role                        | Input Node(s)                    | Output Node(s)                                   | Sticky Note                                                                                  |
|----------------------------|----------------------------|-------------------------------------|---------------------------------|-------------------------------------------------|---------------------------------------------------------------------------------------------|
| 🔔 Linear Trigger           | Linear Trigger             | Entry point; listens for issue create/update events | —                               | 📋 Filter New Issues Only                        | 🔔 Trigger & Filtering block overview                                                     |
| 📋 Filter New Issues Only   | If                         | Filters issues with non-empty title | 🔔 Linear Trigger               | ⚖️ If (Create or Update)                         | 🔔 Trigger & Filtering block overview                                                     |
| ⚖️ If (Create or Update)    | If                         | Allows only create or update actions | 📋 Filter New Issues Only        | 🤖 AI Agent (Bug Classifier)                      | 🔔 Trigger & Filtering block overview                                                     |
| 🧠 OpenAI Chat Model         | Langchain OpenAI Chat LM   | Provides GPT-4-mini LLM for classification | —                               | 🤖 AI Agent (Bug Classifier)                      | 🤖 AI Classification block overview                                                      |
| 🤖 AI Agent (Bug Classifier) | Langchain Agent            | Classifies issue into team ID       | ⚖️ If (Create or Update), 🧠 OpenAI Chat Model | 🔀 Engineering Router, 🔀 Product Router, 🔀 Design Router, 🔀 Default Router | 🤖 AI Classification block overview                                                      |
| 🔀 Engineering Router        | Switch                     | Routes issues classified as Engineering | 🤖 AI Agent (Bug Classifier)    | 🛠️ Assign to Engineering                         | 🔀 Routing Logic block overview                                                          |
| 🔀 Product Router            | Switch                     | Routes issues classified as Product | 🤖 AI Agent (Bug Classifier)    | 📦 Assign to Product                             | 🔀 Routing Logic block overview                                                          |
| 🔀 Design Router             | Switch                     | Routes issues classified as Design  | 🤖 AI Agent (Bug Classifier)    | 🎨 Assign to Design                              | 🔀 Routing Logic block overview                                                          |
| 🔀 Default Router            | Switch                     | Routes issues classified as Default | 🤖 AI Agent (Bug Classifier)    | 📂 Assign to Default                             | 🔀 Routing Logic block overview                                                          |
| 🛠️ Assign to Engineering     | Linear                     | Updates issue team to Engineering   | 🔀 Engineering Router            | —                                               | 🗂️ Assignment Actions block overview                                                     |
| 📦 Assign to Product         | Linear                     | Updates issue team to Product       | 🔀 Product Router                | —                                               | 🗂️ Assignment Actions block overview                                                     |
| 🎨 Assign to Design          | Linear                     | Updates issue team to Design        | 🔀 Design Router                 | —                                               | 🗂️ Assignment Actions block overview                                                     |
| 📂 Assign to Default         | Linear                     | Updates issue team to Default       | 🔀 Default Router                | —                                               | 🗂️ Assignment Actions block overview                                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Linear Trigger node:**  
   - Type: Linear Trigger  
   - Configure with your Linear Team ID (replace `"YOUR_LINEAR_TEAM_ID"`).  
   - Set resource to `issue`.  
   - Assign Linear API credentials.  
   - Position it as the workflow entry point.

2. **Add If node "📋 Filter New Issues Only":**  
   - Type: If  
   - Condition: Check if `data.title` exists and is not empty (`string exists`).  
   - Connect from the Linear Trigger node's main output.

3. **Add If node "⚖️ If (Create or Update)":**  
   - Type: If  
   - Condition: `$json.action` equals `create` OR `update`.  
   - Connect from the Filter New Issues Only node's True branch.

4. **Add Langchain OpenAI Chat Model node "🧠 OpenAI Chat Model":**  
   - Type: Langchain OpenAI Chat LM  
   - Model: Select `gpt-4-mini`.  
   - Assign OpenAI API credentials.

5. **Add Langchain Agent node "🤖 AI Agent (Bug Classifier)":**  
   - Type: Langchain Agent  
   - Prompt:  
     ```
     You are a bug classification expert. Based on the bug title and description, classify it into exactly one of these teams: Engineering, Product, Design, QA, DevOps, Support.

     Rules:
     - If it belongs to Engineering, respond with: ENGINEERING_TEAM_ID
     - If it belongs to Product, respond with: PRODUCT_TEAM_ID
     - If it belongs to Design, respond with: DESIGN_TEAM_ID
     - If it belongs to Nothing(Default), respond with: DEFAULT_TEAM_ID

     Respond with ONLY the correct team ID as specified above. Do not include explanations, text, or formatting.

     Title: {{ $json.data.title }}
     Description: {{ $json.data.description }}
     ```
   - Connect the AI Agent input to the output of the "⚖️ If (Create or Update)" node.  
   - Use the "🧠 OpenAI Chat Model" node as the language model resource for this AI Agent.

6. **Add four Switch nodes for routing:**  
   - Names: "🔀 Engineering Router", "🔀 Product Router", "🔀 Design Router", "🔀 Default Router".  
   - For each, configure the condition on `{{ $json.output }}`:  
     - Engineering Router: equals `ENGINEERING_TEAM_ID`  
     - Product Router: equals `PRODUCT_TEAM_ID`  
     - Design Router: equals `DESIGN_TEAM_ID`  
     - Default Router: equals `DEFAULT_TEAM_ID`  
   - Connect from the AI Agent node's output to all four switch nodes.

7. **Add four Linear update nodes to assign teams:**  
   - Names: "🛠️ Assign to Engineering", "📦 Assign to Product", "🎨 Assign to Design", "📂 Assign to Default".  
   - For each:  
     - Operation: `update`  
     - `issueId`: use expression `{{$('⚖️ If (Create or Update)').item.json.data.id}}` to reference the issue ID.  
     - Update field `teamId` to the corresponding team ID from AI Agent output (use `{{ $json.output }}`).  
     - Credentials: Linear API credentials.  
   - Connect each Linear update node to the corresponding Router node's True output.

8. **Final connections and validation:**  
   - Ensure all nodes are connected as per above.  
   - Validate credentials for Linear and OpenAI nodes.  
   - Test with sample Linear issues to confirm correct routing.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                              | Context or Link                                                                                                           |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------|
| The Linear Trigger node enables real-time automation by listening to specific issue events, allowing immediate processing without delay.                                                                                                                                                                                                                                                | Sticky Note on Trigger & Filtering block                                                                                  |
| The AI classification uses a custom prompt designed to return only the team ID string for deterministic routing, eliminating ambiguity.                                                                                                                                                                                                                                                  | Sticky Note on AI Classification block                                                                                   |
| Routing via switch nodes ensures exactly one assignment path is chosen, preventing duplicate or conflicting updates.                                                                                                                                                                                                                                                                      | Sticky Note on Routing Logic block                                                                                        |
| Linear update nodes directly modify the issue's team assignment, automating the triage workflow and reducing manual workload.                                                                                                                                                                                                                                                             | Sticky Note on Assignment Actions block                                                                                   |
| Replace placeholder team ID strings (e.g., ENGINEERING_TEAM_ID) with your actual Linear team IDs for routing to work correctly.                                                                                                                                                                                                                                                           | Critical setup note                                                                                                       |
| Refer to n8n official documentation for setting up Linear API and OpenAI API credentials to avoid authentication errors.                                                                                                                                                                                                                                                                   | https://docs.n8n.io/integrations/builtin/nodes/n8n-nodes-base.linear/ and https://docs.n8n.io/integrations/builtin/nodes/n8n-nodes-base.openai/ |
| GPT-4-mini is a smaller variant of GPT-4 tailored for cost-effective usage while retaining strong classification capabilities. Use accordingly to balance performance and cost.                                                                                                                                                                                                         | AI model selection note                                                                                                  |

---

**Disclaimer:**  
The provided content is derived exclusively from an n8n automated workflow. The processing strictly adheres to applicable content policies and contains no illegal, offensive, or protected elements. All data handled is lawful and public.