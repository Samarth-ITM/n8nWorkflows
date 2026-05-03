Multi-Source Feedback Processing with OpenAI & Slack Notifications

https://n8nworkflows.xyz/workflows/multi-source-feedback-processing-with-openai---slack-notifications-7784


# Multi-Source Feedback Processing with OpenAI & Slack Notifications

### 1. Workflow Overview

This workflow implements a **Multi-Source Feedback Processing** system designed to accept user feedback from multiple origins—namely a web form, an external webhook, or as input triggered by another workflow. It normalizes and consolidates all incoming data into a unified format, then processes the feedback using OpenAI’s GPT-4 language model to generate a concise summary. Finally, it notifies a Slack channel with the summarized feedback along with its type.

The workflow is logically divided into these blocks:

- **1.1 Input Reception:** Three distinct triggers handle feedback input: a form submission, a webhook POST, and an execution by another workflow.
- **1.2 Data Normalization:** Each input trigger’s output is transformed into a consistent data structure with standardized field names.
- **1.3 Data Consolidation:** All normalized data streams merge into one node to unify the processing flow.
- **1.4 AI Processing:** A chain of nodes that summarize the feedback text using OpenAI’s GPT-4 model.
- **1.5 Notification:** The summarized feedback is sent as a message to a designated Slack channel.
- **1.6 Documentation/Notes:** Several sticky notes provide design pattern explanations and instructions for adapting the workflow.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
This block captures feedback data from three different sources to enable flexible integration options. It supports direct form submissions, webhook calls, and sub-workflow invocations.

**Nodes Involved:**  
- On form submission  
- Webhook  
- When Executed by Another Workflow

**Node Details:**

- **On form submission**  
  - *Type/Role:* Form Trigger — Listens for HTTP form submissions with specified fields.  
  - *Configuration:* Listens for "Product feedback" form with two fields: “Your feedback” (text, required) and “Feedback type” (dropdown with options: Bug, Feature request, Praise, Other, required).  
  - *Expressions/Variables:* Outputs raw form data with keys matching field labels.  
  - *Connections:* Output connected to "Prepare data from form".  
  - *Potential Failures:* Missing required fields, webhook misconfiguration, network issues.

- **Webhook**  
  - *Type/Role:* Webhook Trigger — Accepts POST requests on a defined path.  
  - *Configuration:* HTTP POST webhook listening on path "516b9b23-c30c-4396-a32e-2f139beb3248".  
  - *Expressions/Variables:* Expects JSON body with keys `feedback` and `feedback type`.  
  - *Connections:* Output connected to "Prepare data from webhook".  
  - *Potential Failures:* Invalid or missing JSON body, authentication if required externally, network timeouts.

- **When Executed by Another Workflow**  
  - *Type/Role:* Execute Workflow Trigger — Receives input parameters when called from another workflow.  
  - *Configuration:* Accepts workflow inputs "feedback" and "feedback type".  
  - *Expressions/Variables:* Inputs used directly from calling workflow.  
  - *Connections:* Output connected to "Prepare data from sub-workflow".  
  - *Potential Failures:* Missing input parameters, errors in calling workflow, data type mismatches.

---

#### 1.2 Data Normalization

**Overview:**  
Each input source has a dedicated `Set` node that extracts and restructures the feedback data into a unified format with consistent key names: `feedback` and `feedback type`. This normalization ensures downstream nodes can operate regardless of the original input source.

**Nodes Involved:**  
- Prepare data from form  
- Prepare data from webhook  
- Prepare data from sub-workflow

**Node Details:**

- **Prepare data from form**  
  - *Type/Role:* Set node — Maps form field labels to standardized keys.  
  - *Configuration:* Assigns `feedback = $json['Your feedback']`, `feedback type = $json['Feedback type']`.  
  - *Connections:* Output to "Consolidate trigger data".  
  - *Potential Failures:* If form fields change labels, mappings break; expression evaluation errors.

- **Prepare data from webhook**  
  - *Type/Role:* Set node — Maps webhook body fields to standardized keys.  
  - *Configuration:* Assigns `feedback = $json.body.feedback`, `feedback type = $json.body['feedback type']`.  
  - *Connections:* Output to "Consolidate trigger data".  
  - *Potential Failures:* If webhook payload structure changes or missing keys.

- **Prepare data from sub-workflow**  
  - *Type/Role:* Set node — Maps inputs from sub-workflow execution to standardized keys.  
  - *Configuration:* Assigns `feedback = $json.feedback`, `feedback type = $json['feedback type']`.  
  - *Connections:* Output to "Consolidate trigger data".  
  - *Potential Failures:* Input data missing or malformed.

---

#### 1.3 Data Consolidation

**Overview:**  
This single `Set` node consolidates the normalized data into one standardized object, enabling the workflow to proceed without dependency on the origin of the data.

**Nodes Involved:**  
- Consolidate trigger data

**Node Details:**

- **Consolidate trigger data**  
  - *Type/Role:* Set node — Receives inputs from all normalization nodes, uses generic `$json` expressions to unify data.  
  - *Configuration:* Assigns `feedback = $json.feedback`, `feedback type = $json['feedback type']`.  
  - *Connections:* Output to "Summarise feedback".  
  - *Potential Failures:* If upstream nodes send inconsistent data or unexpected nulls.

---

#### 1.4 AI Processing

**Overview:**  
This block uses OpenAI’s GPT-4 (via Langchain integration) to generate a one-line summary of the feedback. It employs a chain node feeding into a chat model node to produce a concise textual summary.

**Nodes Involved:**  
- Summarise feedback  
- OpenAI Chat Model

**Node Details:**

- **Summarise feedback**  
  - *Type/Role:* Langchain Chain LLM node — Prepares prompt text and sends request to language model.  
  - *Configuration:* Text prompt formatted as:  
    ```
    Feedback:

    {{ $json.feedback }}

    Feedback type:

    {{ $json['feedback type'] }}
    ```  
    Message instruction: "Return a one-line summary of the feedback provided by the user message".  
  - *Connections:* Output to "Notify the team on Slack".  
  - *Potential Failures:* OpenAI API quota limits, prompt formatting errors, timeout.

- **OpenAI Chat Model**  
  - *Type/Role:* Langchain OpenAI Chat Model node — Executes GPT-4.1-mini model call.  
  - *Configuration:* Model set to `gpt-4.1-mini`; credentials linked to OpenAI API key.  
  - *Connections:* Output connected back to "Summarise feedback" node’s language model input.  
  - *Potential Failures:* API key invalid, network errors, rate limits.

---

#### 1.5 Notification

**Overview:**  
After summarizing the feedback, this node sends a notification message to a Slack channel with the summary and feedback type.

**Nodes Involved:**  
- Notify the team on Slack

**Node Details:**

- **Notify the team on Slack**  
  - *Type/Role:* Slack node — Posts message to a Slack channel.  
  - *Configuration:*  
    - Channel: Fixed to channel ID `C025KLW3MQS` (named “général”).  
    - Message text uses expressions to include the summary text and feedback type:  
      ```
      We received a new feedback!
      In summary: {{ $json.text }}

      Type: {{ $('Consolidate trigger data').item.json['feedback type'] }}
      ```  
  - *Credentials:* Uses OAuth2 Slack account credentials.  
  - *Potential Failures:* Invalid Slack token, channel permissions, network errors.

---

#### 1.6 Documentation / Notes

**Overview:**  
Multiple sticky notes provide detailed explanations of the workflow design pattern used and instructions for adapting the workflow.

**Nodes Involved:**  
- Sticky Note  
- Sticky Note1  
- Sticky Note2  
- Sticky Note3  
- Sticky Note4

**Node Details:**

- **Sticky Note**  
  - Explains the "Multi-Trigger Unification Pattern" that enables multiple input sources to feed the same processing logic by normalizing and consolidating data.  
  - Contains instructions on how to replace triggers and prepare normalization nodes, emphasizing maintainability and extensibility.  
  - Credits Guillaume Duvernay for the template.

- **Sticky Note1**  
  - Brief note: "Replace with the workflow triggers you need".

- **Sticky Note2**  
  - Reminder to prepare variables ahead of consolidation, ensuring all normalization nodes use the same field names.

- **Sticky Note3**  
  - Indicates that the consolidation node is the point to build the main workflow logic afterward.

- **Sticky Note4**  
  - Describes the consolidation node’s role in merging data from various trigger paths.

---

### 3. Summary Table

| Node Name                  | Node Type                                         | Functional Role                      | Input Node(s)               | Output Node(s)                | Sticky Note                                                                                                                        |
|----------------------------|--------------------------------------------------|------------------------------------|-----------------------------|------------------------------|-----------------------------------------------------------------------------------------------------------------------------------|
| On form submission          | Form Trigger                                     | Feedback input via web form         | —                           | Prepare data from form         | Replace with the workflow triggers you need                                                                                       |
| Prepare data from form      | Set Node                                         | Normalize form data                 | On form submission           | Consolidate trigger data       | Prepare the variables before the consolidation. Make sure to use the same field names to be able to consolidate them later!       |
| When Executed by Another Workflow | Execute Workflow Trigger                     | Feedback input from sub-workflow   | —                           | Prepare data from sub-workflow | Replace with the workflow triggers you need                                                                                       |
| Prepare data from sub-workflow | Set Node                                      | Normalize sub-workflow input       | When Executed by Another Workflow | Consolidate trigger data       | Prepare the variables before the consolidation. Make sure to use the same field names to be able to consolidate them later!       |
| Webhook                    | Webhook Trigger                                  | Feedback input via webhook          | —                           | Prepare data from webhook      | Replace with the workflow triggers you need                                                                                       |
| Prepare data from webhook   | Set Node                                         | Normalize webhook data              | Webhook                     | Consolidate trigger data       | Prepare the variables before the consolidation. Make sure to use the same field names to be able to consolidate them later!       |
| Consolidate trigger data    | Set Node                                         | Consolidate normalized data        | Prepare data from form, Prepare data from webhook, Prepare data from sub-workflow | Summarise feedback             | Consolidation node. This takes data from the previous nodes, no matter the path. Map previous nodes to this consolidation node.  |
| Summarise feedback          | Langchain Chain LLM                              | Generate one-line feedback summary | Consolidate trigger data      | Notify the team on Slack       | Replace this and build your workflow                                                                                              |
| OpenAI Chat Model           | Langchain OpenAI Chat Model                      | GPT-4 language model execution     | Summarise feedback (as LM input) | Summarise feedback (LLM output) | Replace this and build your workflow                                                                                              |
| Notify the team on Slack    | Slack Node                                       | Send notification to Slack channel | Summarise feedback           | —                            |                                                                                                                                   |
| Sticky Note                 | Sticky Note                                      | Documentation & pattern explanation | —                           | —                            | Multi-Trigger Unification Pattern explanation and adaptation instructions. Template by Guillaume Duvernay                       |
| Sticky Note1                | Sticky Note                                      | Documentation                      | —                           | —                            | Replace with the workflow triggers you need                                                                                       |
| Sticky Note2                | Sticky Note                                      | Documentation                      | —                           | —                            | Prepare the variables before the consolidation. Make sure to use the same field names to be able to consolidate them later!       |
| Sticky Note3                | Sticky Note                                      | Documentation                      | —                           | —                            | Replace this and build your workflow                                                                                              |
| Sticky Note4                | Sticky Note                                      | Documentation                      | —                           | —                            | Consolidation node. This takes data from the previous nodes, no matter the path. Map previous nodes to this consolidation node.  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes:**

   a. *On form submission*  
      - Add a **Form Trigger** node.  
      - Set form title: "Product feedback".  
      - Add two fields:  
        - "Your feedback" (Text, Required)  
        - "Feedback type" (Dropdown, options: Bug, Feature request, Praise, Other, Required)  
      - Save.

   b. *Webhook*  
      - Add a **Webhook** node.  
      - Set HTTP Method: POST.  
      - Set Path: A unique string (e.g., `516b9b23-c30c-4396-a32e-2f139beb3248`).  
      - Save.

   c. *When Executed by Another Workflow*  
      - Add an **Execute Workflow Trigger** node.  
      - Define input parameters: "feedback" (string), "feedback type" (string).  
      - Save.

2. **Create Normalization Nodes (Set nodes):**

   a. *Prepare data from form*  
      - Add a **Set** node.  
      - Assign two variables:  
        - `feedback` = `{{$json["Your feedback"]}}`  
        - `feedback type` = `{{$json["Feedback type"]}}`  
      - Connect output from *On form submission* trigger.

   b. *Prepare data from webhook*  
      - Add a **Set** node.  
      - Assign:  
        - `feedback` = `{{$json.body.feedback}}`  
        - `feedback type` = `{{$json.body["feedback type"]}}`  
      - Connect output from *Webhook* trigger.

   c. *Prepare data from sub-workflow*  
      - Add a **Set** node.  
      - Assign:  
        - `feedback` = `{{$json.feedback}}`  
        - `feedback type` = `{{$json["feedback type"]}}`  
      - Connect output from *When Executed by Another Workflow* trigger.

3. **Create Consolidation Node:**

   - Add a **Set** node named "Consolidate trigger data".  
   - Assign:  
     - `feedback` = `{{$json.feedback}}`  
     - `feedback type` = `{{$json["feedback type"]}}`  
   - Connect outputs from all three normalization nodes to this node (merge all paths here).

4. **Create AI Processing Nodes:**

   a. *Summarise feedback*  
      - Add a **Langchain Chain LLM** node.  
      - Set prompt text:  
        ```
        Feedback:

        {{ $json.feedback }}

        Feedback type:

        {{ $json["feedback type"] }}
        ```  
      - Add message instruction: "Return a one-line summary of the feedback provided by the user message".  
      - Connect output of "Consolidate trigger data" node.

   b. *OpenAI Chat Model*  
      - Add a **Langchain OpenAI Chat Model** node.  
      - Select model: `gpt-4.1-mini`.  
      - Link OpenAI API credentials (set up credentials with an OpenAI API key).  
      - Connect as the language model input for the "Summarise feedback" node.

5. **Create Notification Node:**

   - Add a **Slack** node.  
   - Configure to send messages to a specific channel:  
     - Channel ID: `C025KLW3MQS` (replace with your actual Slack channel ID).  
   - Message text:  
     ```
     We received a new feedback!
     In summary: {{ $json.text }}

     Type: {{ $('Consolidate trigger data').item.json["feedback type"] }}
     ```  
   - Link Slack credentials (OAuth2, set up Slack app and token).  
   - Connect output from "Summarise feedback" node.

6. **Add Sticky Notes for Documentation (optional but recommended):**

   - Add notes explaining the multi-trigger unification pattern, normalization, consolidation, and where to customize triggers or processing.

7. **Test the workflow:**

   - Submit feedback via the form, webhook, and sub-workflow invocation to verify consistent processing and Slack notification delivery.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                              | Context or Link                                                                                   |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| This workflow demonstrates the "Multi-Trigger Unification Pattern" which normalizes diverse input sources into a single processing pipeline, improving maintainability and enabling easy extension to new triggers.                          | Explanation inside main Sticky Note node                                                        |
| Template and pattern credited to Guillaume Duvernay.                                                                                                                                                                                     | Template author                                                                                  |
| Slack channel ID `C025KLW3MQS` corresponds to the "général" channel. Replace with your channel’s ID as needed.                                                                                                                          | Slack integration configuration                                                                 |
| OpenAI model used is `gpt-4.1-mini` via Langchain’s n8n integration. Make sure your OpenAI API key has access to GPT-4 models and sufficient quota.                                                                                     | OpenAI API integration                                                                          |
| To adapt this workflow, replace trigger nodes with your input sources and update the corresponding normalization Set nodes to use consistent field names. Build your custom logic after the consolidation node.                         | Adaptation instructions from Sticky Notes                                                      |

---

**Disclaimer:**  
The provided text originates solely from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data processed is legal and public.