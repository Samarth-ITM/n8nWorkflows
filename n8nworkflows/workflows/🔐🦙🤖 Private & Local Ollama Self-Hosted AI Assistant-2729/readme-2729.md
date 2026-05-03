🔐🦙🤖 Private & Local Ollama Self-Hosted AI Assistant

https://n8nworkflows.xyz/workflows/-------private---local-ollama-self-hosted-ai-assistant-2729


# 🔐🦙🤖 Private & Local Ollama Self-Hosted AI Assistant

### 1. Workflow Overview

This workflow, titled **"🔐🦙🤖 Private & Local Ollama Self-Hosted AI Assistant"**, transforms a local n8n instance into a private, self-hosted AI chat interface using the Ollama language model platform. It is designed to process chat messages locally with zero cloud dependencies, leveraging the Llama 3.2 model (or any Ollama-compatible model) to generate structured JSON responses.

The workflow is organized into the following logical blocks:

- **1.1 Input Reception:** Captures incoming chat messages via a LangChain chat trigger node.
- **1.2 AI Processing:** Processes the chat input through a LangChain LLM chain node, which uses the Ollama Llama 3.2 model to generate a JSON-formatted response.
- **1.3 Response Structuring:** Transforms the raw model output into a structured JSON object and formats the final response text.
- **1.4 Error Handling:** Provides fallback messaging in case the AI processing fails, ensuring robustness.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block initiates the workflow by listening for incoming chat messages. It acts as the entry point for user interaction.

- **Nodes Involved:**  
  - When chat message received

- **Node Details:**  
  - **When chat message received**  
    - Type: LangChain Chat Trigger  
    - Role: Listens for new chat messages to trigger the workflow.  
    - Configuration: Default options; no special filters or conditions.  
    - Inputs: External chat message webhook trigger.  
    - Outputs: Passes chat input to the Basic LLM Chain node.  
    - Version: 1.1  
    - Edge Cases: Possible webhook connectivity issues or malformed incoming messages.  
    - Sticky Note: Describes the trigger node’s purpose as the workflow initiator.

#### 2.2 AI Processing

- **Overview:**  
  This block processes the chat input using a LangChain LLM chain node configured with a prompt that instructs the model to return a JSON object containing the prompt and response. The Ollama model node provides the language model backend.

- **Nodes Involved:**  
  - Basic LLM Chain  
  - Ollama Model

- **Node Details:**  
  - **Basic LLM Chain**  
    - Type: LangChain LLM Chain  
    - Role: Processes the chat input with a custom prompt and calls the language model.  
    - Configuration:  
      - Prompt: Instructs the model to output a JSON object with fields "Prompt" and "Response" based on the user's chat input (`{{ $json.chatInput }}`).  
      - On error: Continues to error output branch.  
    - Inputs: Receives chat input from the trigger node.  
    - Outputs: Sends model output text to JSON to Object node on success; error output to Error Response node.  
    - Version: 1.5  
    - Edge Cases: Model response may not be valid JSON; timeout or API errors; expression evaluation failures.  
    - Sticky Notes: Explains the processing node’s role and prompt details.

  - **Ollama Model**  
    - Type: LangChain Ollama Integration  
    - Role: Provides the Llama 3.2 language model backend for the LLM chain.  
    - Configuration:  
      - Model: `llama3.2:latest` (default Ollama model)  
      - Options: Default (empty)  
      - Credentials: Uses configured Ollama API credentials.  
    - Inputs: Connected as the language model for the Basic LLM Chain node.  
    - Outputs: Provides model-generated text to the LLM chain.  
    - Version: 1  
    - Edge Cases: Authentication errors, model unavailability, API timeouts.  
    - Sticky Note: Describes the model node’s purpose and configuration.

#### 2.3 Response Structuring

- **Overview:**  
  This block converts the raw text output from the AI model into a structured JSON object, then formats a user-friendly response string.

- **Nodes Involved:**  
  - JSON to Object  
  - Structured Response

- **Node Details:**  
  - **JSON to Object**  
    - Type: Set Node  
    - Role: Parses the raw text output from the Basic LLM Chain into a JSON object under the field `response`.  
    - Configuration:  
      - Manual mapping mode  
      - Assigns `response` field from the JSON-parsed `text` property of the previous node.  
    - Inputs: Receives raw text from Basic LLM Chain.  
    - Outputs: Passes structured JSON to Structured Response node.  
    - Version: 3.4  
    - Edge Cases: Invalid JSON parsing if model output is malformed.  
    - Sticky Note: Details the purpose of manual mapping and JSON transformation.

  - **Structured Response**  
    - Type: Set Node  
    - Role: Formats the final response text combining the prompt and response fields from the JSON object, and includes the original JSON object for transparency.  
    - Configuration:  
      - Manual mapping mode  
      - Sets `text` field with a template string referencing `response.Prompt`, `response.Response`, and the original JSON text from Basic LLM Chain.  
    - Inputs: Receives structured JSON from JSON to Object node.  
    - Outputs: Final output of the workflow (chat response).  
    - Version: 3.4  
    - Edge Cases: Missing fields in JSON object may cause empty or malformed output.  
    - Sticky Note: Explains the response formatting and manual mapping.

#### 2.4 Error Handling

- **Overview:**  
  This block ensures that if the Basic LLM Chain fails (e.g., due to model errors or invalid input), the workflow returns a default error message instead of failing silently.

- **Nodes Involved:**  
  - Error Response

- **Node Details:**  
  - **Error Response**  
    - Type: Set Node  
    - Role: Provides a fallback text response indicating an error occurred during processing.  
    - Configuration:  
      - Sets `text` field to a static message: "There was an error."  
    - Inputs: Connected to the error output branch of Basic LLM Chain.  
    - Outputs: Returns error message as workflow output.  
    - Version: 3.4  
    - Edge Cases: Should always succeed; minimal failure risk.  
    - Sticky Note: Describes error handling purpose and connection to LLM chain error output.

---

### 3. Summary Table

| Node Name               | Node Type                      | Functional Role                          | Input Node(s)            | Output Node(s)          | Sticky Note                                                                                  |
|-------------------------|--------------------------------|----------------------------------------|--------------------------|-------------------------|----------------------------------------------------------------------------------------------|
| When chat message received | LangChain Chat Trigger         | Entry point; receives chat messages    | External webhook         | Basic LLM Chain          | ## Trigger Node - Initiates workflow on new chat message                                   |
| Basic LLM Chain          | LangChain LLM Chain            | Processes input with AI prompt         | When chat message received | JSON to Object, Error Response | ## Processing Node - Handles message processing and returns structured JSON                 |
| Ollama Model            | LangChain Ollama Integration    | Provides Llama 3.2 model backend       | Basic LLM Chain (ai_languageModel) | Basic LLM Chain (ai_languageModel) | ## Model Node - Provides language model capabilities                                       |
| JSON to Object           | Set Node                      | Parses raw AI text output to JSON      | Basic LLM Chain          | Structured Response      | ## JSON to Object Node - Transforms response data into structured JSON                      |
| Structured Response      | Set Node                      | Formats final response text             | JSON to Object           | (Workflow output)        | ## Structured Response Node - Formats chat response with prompt and AI reply                |
| Error Response           | Set Node                      | Provides fallback error message         | Basic LLM Chain (error)  | (Workflow output)        | ## Error Response Node - Handles errors from LLM Chain, returns default error message       |
| Sticky Note              | Sticky Note                   | Documentation and notes                  |                          |                         | # 🦙 Ollama Chat Workflow overview and setup instructions                                  |
| Sticky Note1             | Sticky Note                   | Model node description                   |                          |                         | ## Model Node - Details on Ollama Model node                                               |
| Sticky Note2             | Sticky Note                   | Trigger node description                 |                          |                         | ## Trigger Node - Describes chat message trigger                                          |
| Sticky Note3             | Sticky Note                   | Processing node description              |                          |                         | ## Processing Node - Explains Basic LLM Chain node                                        |
| Sticky Note4             | Sticky Note                   | Prompt details                           |                          |                         | ### Prompt (Change this for your use case)                                                |
| Sticky Note5             | Sticky Note                   | JSON to Object node description          |                          |                         | ## JSON to Object Node - Manual mapping and JSON transformation                           |
| Sticky Note6             | Sticky Note                   | Structured Response node description     |                          |                         | ## Structured Response Node - Response formatting details                                |
| Sticky Note7             | Sticky Note                   | Error Response node description          |                          |                         | ## Error Response Node - Error handling and fallback messaging                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node:**  
   - Add a **LangChain Chat Trigger** node named `When chat message received`.  
   - Use default settings to listen for incoming chat messages.  
   - This node will serve as the workflow entry point.

2. **Create AI Processing Chain:**  
   - Add a **LangChain LLM Chain** node named `Basic LLM Chain`.  
   - Configure the prompt as:  
     ```
     Provide the users prompt and response as a JSON object with two fields:
     - Prompt
     - Response

     Avoid any preample or further explanation.

     This is the question: {{ $json.chatInput }}
     ```  
   - Set **On Error** to `continueErrorOutput` to enable error branch.  
   - Connect the output of `When chat message received` to the input of this node.

3. **Configure Ollama Model Node:**  
   - Add a **LangChain Ollama** node named `Ollama Model`.  
   - Set the model to `llama3.2:latest`.  
   - Leave options empty (default).  
   - Assign valid **Ollama API credentials** (must be pre-configured in n8n).  
   - Connect this node as the language model for the `Basic LLM Chain` node (via the `ai_languageModel` input).

4. **Add JSON Parsing Node:**  
   - Add a **Set** node named `JSON to Object`.  
   - Configure manual mapping mode.  
   - Add an assignment:  
     - Field Name: `response`  
     - Type: Object  
     - Value: `={{ $json.text }}` (parses the raw text output from the LLM chain into JSON).  
   - Connect the main output of `Basic LLM Chain` to this node.

5. **Add Structured Response Node:**  
   - Add a **Set** node named `Structured Response`.  
   - Configure manual mapping mode.  
   - Add an assignment:  
     - Field Name: `text`  
     - Type: String  
     - Value:  
       ```
       Your prompt was: {{ $json.response.Prompt }}

       My response is: {{ $json.response.Response }}

       This is the JSON object:

       {{ $('Basic LLM Chain').item.json.text }}
       ```  
   - Connect the output of `JSON to Object` to this node.

6. **Add Error Handling Node:**  
   - Add a **Set** node named `Error Response`.  
   - Configure manual mapping mode.  
   - Add an assignment:  
     - Field Name: `text`  
     - Type: String  
     - Value: `There was an error.`  
   - Connect the error output branch of `Basic LLM Chain` to this node.

7. **Finalize Connections:**  
   - Connect the main output of `Structured Response` as the workflow’s final output.  
   - Connect the output of `Error Response` as an alternative final output for error cases.

8. **Activate Workflow:**  
   - Ensure all nodes are properly connected and credentials are valid.  
   - Activate the workflow to start processing incoming chat messages.

---

### 5. General Notes & Resources

| Note Content                                                                                                    | Context or Link                                                                                   |
|----------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| This workflow enables fully local, private AI chat processing using Ollama models, avoiding cloud dependencies. | Workflow description and purpose                                                                |
| Setup requires installing n8n and Ollama, downloading the Llama 3.2 model, and configuring Ollama API credentials. | Setup instructions                                                                              |
| The prompt in the Basic LLM Chain node can be customized to fit different use cases or response formats.        | Prompt customization note (Sticky Note4)                                                        |
| For more information on Ollama and Llama models, visit https://ollama.com/                                    | Official Ollama website                                                                          |
| The workflow includes robust error handling to ensure users receive feedback even if AI processing fails.       | Error handling design note (Sticky Note7)                                                       |

---

This documentation provides a detailed, structured reference for understanding, reproducing, and modifying the "Private & Local Ollama Self-Hosted AI Assistant" workflow in n8n. It covers all nodes, their roles, configurations, and interconnections, enabling both advanced users and AI agents to work effectively with this workflow.