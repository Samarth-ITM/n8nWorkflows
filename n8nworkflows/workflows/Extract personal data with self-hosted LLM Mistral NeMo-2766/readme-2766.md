Extract personal data with self-hosted LLM Mistral NeMo

https://n8nworkflows.xyz/workflows/extract-personal-data-with-self-hosted-llm-mistral-nemo-2766


# Extract personal data with self-hosted LLM Mistral NeMo

### 1. Workflow Overview

This workflow demonstrates how to extract structured personal data from unstructured user input using a self-hosted Large Language Model (LLM), specifically the Mistral NeMo model running locally via Ollama. It leverages n8n’s LangChain integration to process chat messages, parse the output into a defined JSON schema, and implement error handling with auto-fixing capabilities to ensure data consistency.

**Target Use Cases:**  
- Enterprise environments requiring local data processing for privacy compliance  
- Automated extraction of personal information (e.g., name, contact method) from free-text inputs  
- Scenarios needing structured JSON output from conversational AI responses  

**Logical Blocks:**  
- **1.1 Input Reception:** Receives chat messages triggering the workflow  
- **1.2 LLM Processing:** Sends input to the local Mistral NeMo model via Ollama and processes responses  
- **1.3 Output Parsing and Validation:** Parses LLM output into a structured JSON format and validates it against a schema  
- **1.4 Auto-fixing and Error Handling:** Automatically retries and corrects outputs that do not meet schema constraints  
- **1.5 Final Output Preparation:** Extracts and formats the final JSON output for downstream use  

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
This block listens for incoming chat messages that trigger the workflow to start processing.

**Nodes Involved:**  
- When chat message received

**Node Details:**  
- **When chat message received**  
  - Type: `@n8n/n8n-nodes-langchain.chatTrigger`  
  - Role: Entry point webhook node that triggers the workflow upon receiving a chat message  
  - Configuration: Default webhook with no special options; listens for chat input  
  - Inputs: External chat message via webhook  
  - Outputs: Passes user input to the Basic LLM Chain node  
  - Edge Cases: Webhook connectivity issues, malformed or empty chat messages  
  - Version: 1.1  

---

#### 1.2 LLM Processing

**Overview:**  
This block sends the user input to the self-hosted Mistral NeMo LLM via Ollama, using a Basic LLM Chain with system prompts to instruct the model on extracting data.

**Nodes Involved:**  
- Basic LLM Chain  
- Ollama Chat Model

**Node Details:**  
- **Basic LLM Chain**  
  - Type: `@n8n/n8n-nodes-langchain.chainLlm`  
  - Role: Constructs the prompt and manages the LLM chain execution  
  - Configuration:  
    - Message prompt includes instruction to analyze user request and extract info per JSON schema  
    - Dynamic date injected with expression `{{ $now.toISO() }}`  
    - Output parser enabled to validate structured output  
    - On error: continues with error output for downstream handling  
  - Inputs: Receives chat message from trigger node  
  - Outputs: Sends output to "Extract JSON Output" and "On Error" nodes  
  - Edge Cases: Expression evaluation failures, prompt formatting errors, LLM response delays  
  - Version: 1.5  

- **Ollama Chat Model**  
  - Type: `@n8n/n8n-nodes-langchain.lmChatOllama`  
  - Role: Connects to the local Mistral NeMo model via Ollama API  
  - Configuration:  
    - Model: `mistral-nemo:latest`  
    - Options: `useMLock` enabled for memory locking, `keepAlive` set to 2 hours, low temperature (0.1) for deterministic output  
  - Credentials: Uses stored Ollama API credentials  
  - Inputs: Connected as AI language model for both Basic LLM Chain and Auto-fixing Output Parser  
  - Outputs: Provides LLM completions to downstream nodes  
  - Edge Cases: API authentication errors, model unavailability, timeout, resource constraints  
  - Version: 1  

---

#### 1.3 Output Parsing and Validation

**Overview:**  
This block parses the raw LLM output into a structured JSON format based on a manually defined JSON schema and validates the output.

**Nodes Involved:**  
- Structured Output Parser

**Node Details:**  
- **Structured Output Parser**  
  - Type: `@n8n/n8n-nodes-langchain.outputParserStructured`  
  - Role: Parses and validates LLM output against a JSON schema  
  - Configuration:  
    - Schema manually defined with properties: `name`, `surname`, `commtype` (enum: email, phone, other), `contacts`, `timestamp` (date-time), `subject`  
    - Required fields: `name`, `commtype`  
  - Inputs: Receives raw LLM output from Auto-fixing Output Parser  
  - Outputs: Sends parsed output to Auto-fixing Output Parser for validation and potential correction  
  - Edge Cases: Schema validation failures, missing required fields, format errors  
  - Version: 1.2  

---

#### 1.4 Auto-fixing and Error Handling

**Overview:**  
If the LLM output does not satisfy the schema constraints, this block automatically retries the LLM call with an adjusted prompt to fix errors, ensuring output compliance.

**Nodes Involved:**  
- Auto-fixing Output Parser  
- On Error

**Node Details:**  
- **Auto-fixing Output Parser**  
  - Type: `@n8n/n8n-nodes-langchain.outputParserAutofixing`  
  - Role: Wraps the output parser to detect errors and automatically retry with corrective prompts  
  - Configuration:  
    - Custom prompt template instructing the model to fix errors based on instructions, completion, and error details  
    - Only returns corrected output that satisfies constraints  
  - Inputs: Receives output from Structured Output Parser and uses Ollama Chat Model for retries  
  - Outputs: Sends corrected output to Basic LLM Chain for final processing  
  - Edge Cases: Infinite retry loops if model cannot fix errors, prompt misinterpretation, API errors during retry  
  - Version: 1  

- **On Error**  
  - Type: `n8n-nodes-base.noOp`  
  - Role: Placeholder node to handle errors gracefully without stopping workflow execution  
  - Inputs: Receives error output from Basic LLM Chain  
  - Outputs: None  
  - Edge Cases: None (no operation)  
  - Version: 1  

---

#### 1.5 Final Output Preparation

**Overview:**  
This block extracts the final JSON output from the LLM chain response and prepares it for downstream consumption or storage.

**Nodes Involved:**  
- Extract JSON Output

**Node Details:**  
- **Extract JSON Output**  
  - Type: `n8n-nodes-base.set`  
  - Role: Extracts the `output` property from the LLM chain response JSON and sets it as the node’s output  
  - Configuration:  
    - Mode: Raw  
    - Expression: `={{ $json.output }}` to extract the parsed JSON output  
  - Inputs: Receives output from Basic LLM Chain  
  - Outputs: Final structured JSON data for further processing or storage  
  - Edge Cases: Missing or malformed `output` property, empty JSON  
  - Version: 3.4  

---

### 3. Summary Table

| Node Name                 | Node Type                                   | Functional Role                         | Input Node(s)               | Output Node(s)                       | Sticky Note                                                                                     |
|---------------------------|---------------------------------------------|---------------------------------------|-----------------------------|------------------------------------|------------------------------------------------------------------------------------------------|
| When chat message received| @n8n/n8n-nodes-langchain.chatTrigger        | Entry point webhook for chat input    | -                           | Basic LLM Chain                    |                                                                                                |
| Basic LLM Chain           | @n8n/n8n-nodes-langchain.chainLlm           | Constructs prompt and manages LLM chain| When chat message received   | Extract JSON Output, On Error      | Update data source: When you change the data source, update the `Prompt Source (User Message)` setting in this node. |
| Extract JSON Output       | n8n-nodes-base.set                           | Extracts final JSON output             | Basic LLM Chain              | -                                  | When the LLM model responds, the output is checked in the **Structured Output Parser**          |
| On Error                  | n8n-nodes-base.noOp                          | Handles errors gracefully              | Basic LLM Chain (error path) | -                                  |                                                                                                |
| Ollama Chat Model         | @n8n/n8n-nodes-langchain.lmChatOllama       | Connects to local Mistral NeMo model  | -                           | Auto-fixing Output Parser, Basic LLM Chain | Configure local LLM: Ollama offers additional settings to optimize model performance or memory usage. |
| Auto-fixing Output Parser | @n8n/n8n-nodes-langchain.outputParserAutofixing | Auto-fixes invalid LLM outputs         | Structured Output Parser, Ollama Chat Model | Basic LLM Chain                    | If the LLM response does not pass the **Structured Output Parser** checks, **Auto-Fixer** retries with a different prompt. |
| Structured Output Parser  | @n8n/n8n-nodes-langchain.outputParserStructured | Parses and validates LLM output JSON   | Auto-fixing Output Parser    | Auto-fixing Output Parser          | Define JSON Schema                                                                            |
| Sticky Note               | n8n-nodes-base.stickyNote                    | Informational notes                    | -                           | -                                  | See individual sticky note contents below                                                    |
| Sticky Note1              | n8n-nodes-base.stickyNote                    | Informational notes                    | -                           | -                                  | Configure local LLM: Ollama offers additional settings to optimize model performance or memory usage. |
| Sticky Note2              | n8n-nodes-base.stickyNote                    | Informational notes                    | -                           | -                                  | Define JSON Schema                                                                            |
| Sticky Note3              | n8n-nodes-base.stickyNote                    | Informational notes                    | -                           | -                                  | If the LLM response does not pass the **Structured Output Parser** checks, **Auto-Fixer** retries with a different prompt. |
| Sticky Note6              | n8n-nodes-base.stickyNote                    | Informational notes                    | -                           | -                                  | The same LLM connects to both **Basic LLM Chain** and to the **Auto-fixing Output Parser**.   |
| Sticky Note7              | n8n-nodes-base.stickyNote                    | Informational notes                    | -                           | -                                  | When the LLM model responds, the output is checked in the **Structured Output Parser**         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Trigger Node:**  
   - Add a **When chat message received** node (`@n8n/n8n-nodes-langchain.chatTrigger`)  
   - Configure it with default webhook settings to listen for incoming chat messages.

2. **Add the Ollama Chat Model Node:**  
   - Add an **Ollama Chat Model** node (`@n8n/n8n-nodes-langchain.lmChatOllama`)  
   - Set model to `mistral-nemo:latest`  
   - Enable `useMLock` to true  
   - Set `keepAlive` to `2h`  
   - Set `temperature` to `0.1` for deterministic output  
   - Configure credentials with your Ollama API account.

3. **Create the Structured Output Parser Node:**  
   - Add a **Structured Output Parser** node (`@n8n/n8n-nodes-langchain.outputParserStructured`)  
   - Set `schemaType` to `manual`  
   - Define the JSON schema as:  
     ```json
     {
       "type": "object",
       "properties": {
         "name": { "type": "string", "description": "Name of the user" },
         "surname": { "type": "string", "description": "Surname of the user" },
         "commtype": { "type": "string", "enum": ["email", "phone", "other"], "description": "Method of communication" },
         "contacts": { "type": "string", "description": "Contact details. ONLY IF PROVIDED" },
         "timestamp": { "type": "string", "format": "date-time", "description": "When the communication occurred" },
         "subject": { "type": "string", "description": "Brief description of the communication topic" }
       },
       "required": ["name", "commtype"]
     }
     ```

4. **Add the Auto-fixing Output Parser Node:**  
   - Add an **Auto-fixing Output Parser** node (`@n8n/n8n-nodes-langchain.outputParserAutofixing`)  
   - Configure the prompt template to:  
     ```
     Instructions:
     --------------
     {instructions}
     --------------
     Completion:
     --------------
     {completion}
     --------------
     
     Above, the Completion did not satisfy the constraints given in the Instructions.
     Error:
     --------------
     {error}
     --------------
     
     Please try again. Please only respond with an answer that satisfies the constraints laid out in the Instructions:
     ```
   - Connect its AI language model input to the Ollama Chat Model node.  
   - Connect its AI output parser input to the Structured Output Parser node.

5. **Create the Basic LLM Chain Node:**  
   - Add a **Basic LLM Chain** node (`@n8n/n8n-nodes-langchain.chainLlm`)  
   - Configure the message prompt to:  
     ```
     Please analyse the incoming user request. Extract information according to the JSON schema. Today is: "{{ $now.toISO() }}"
     ```  
   - Enable output parser usage.  
   - Set on error behavior to continue with error output.  
   - Connect its AI language model input to the Ollama Chat Model node.  
   - Connect its AI output parser input to the Auto-fixing Output Parser node.  
   - Connect the main input to the **When chat message received** node.

6. **Add the Extract JSON Output Node:**  
   - Add a **Set** node (`n8n-nodes-base.set`) named "Extract JSON Output"  
   - Set mode to `raw`  
   - Add a JSON output expression: `={{ $json.output }}` to extract the parsed output from the Basic LLM Chain response.  
   - Connect its input to the Basic LLM Chain node’s main output.

7. **Add the On Error Node:**  
   - Add a **NoOp** node (`n8n-nodes-base.noOp`) named "On Error"  
   - Connect the error output of the Basic LLM Chain node to this node to handle errors gracefully.

8. **Connect Nodes Appropriately:**  
   - Connect **When chat message received** → **Basic LLM Chain**  
   - Connect **Basic LLM Chain** → **Extract JSON Output** (main output)  
   - Connect **Basic LLM Chain** (error output) → **On Error**  
   - Connect **Ollama Chat Model** → AI language model inputs of **Basic LLM Chain** and **Auto-fixing Output Parser**  
   - Connect **Structured Output Parser** → AI output parser input of **Auto-fixing Output Parser**  
   - Connect **Auto-fixing Output Parser** → AI output parser input of **Basic LLM Chain**

9. **Credential Setup:**  
   - Ensure Ollama API credentials are configured in n8n and linked to the Ollama Chat Model node.

10. **Testing and Validation:**  
    - Trigger the workflow by sending a chat message to the webhook URL.  
    - Verify that the output JSON matches the schema and that errors trigger auto-fixing retries.

---

### 5. General Notes & Resources

| Note Content                                                                                                  | Context or Link                                                                                  |
|---------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| When you change the data source, remember to update the `Prompt Source (User Message)` setting in the Basic LLM Chain node. | Sticky Note on Basic LLM Chain node                                                             |
| Ollama offers additional settings to optimize model performance or memory usage.                              | Sticky Note near Ollama Chat Model node                                                         |
| If the LLM response does not pass the Structured Output Parser checks, Auto-Fixer will call the model again with a different prompt to correct the original response. | Sticky Note near Auto-fixing Output Parser node                                                 |
| The same LLM connects to both Basic LLM Chain and to the Auto-fixing Output Parser.                           | Sticky Note near Ollama Chat Model and Auto-fixing Output Parser nodes                          |
| When the LLM model responds, the output is checked in the Structured Output Parser.                           | Sticky Note near Extract JSON Output and Structured Output Parser nodes                         |
| Comprehensive guide on open-source LLMs with n8n: [https://blog.n8n.io/open-source-llm/](https://blog.n8n.io/open-source-llm/) | Documentation link provided in workflow description                                            |
| Run LLMs locally with n8n: [https://blog.n8n.io/local-llm/](https://blog.n8n.io/local-llm/)                    | Additional resource link                                                                        |
| Video tutorial on using local AI with n8n: [https://www.youtube.com/watch?v=xz_X2N-hPg0](https://www.youtube.com/watch?v=xz_X2N-hPg0) | Video tutorial link                                                                            |

---

This structured documentation provides a complete understanding of the workflow’s architecture, node configurations, error handling strategies, and step-by-step instructions to reproduce or modify the workflow in n8n.