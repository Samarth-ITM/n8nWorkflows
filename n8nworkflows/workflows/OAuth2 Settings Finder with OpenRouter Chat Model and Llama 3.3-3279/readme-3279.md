OAuth2 Settings Finder with OpenRouter Chat Model and Llama 3.3

https://n8nworkflows.xyz/workflows/oauth2-settings-finder-with-openrouter-chat-model-and-llama-3-3-3279


# OAuth2 Settings Finder with OpenRouter Chat Model and Llama 3.3

### 1. Workflow Overview

This workflow, titled **"OAuth2 Settings Finder with OpenRouter Chat Model and Llama 3.3"**, is designed to automate the extraction and validation of OAuth2 configuration details for various API services using AI-powered natural language processing. Its primary use case is to assist developers, IT professionals, and API integrators in quickly obtaining OAuth2 endpoints such as the authorization URI, token URI, and audience parameter based on a simple service name input.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Accepts input data from another workflow, typically a JSON object containing the OAuth service name and optionally the audience.
- **1.2 AI Processing:** Sends the input prompt to an advanced AI language model (Wayfarer Large 70b Llama 3.3 via OpenRouter) to infer OAuth2 settings, including confidence scoring.
- **1.3 Output Parsing:** Parses the AI model’s raw output into a structured JSON format using a predefined schema.
- **1.4 Output Conformation:** Uses a Code node to transform and normalize the parsed output into a consistent structure expected by downstream processes.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block triggers the workflow execution and provides the initial input JSON containing the OAuth service name (e.g., "Atlassian") and optionally the audience. It serves as the entry point for the workflow when invoked by another workflow.

- **Nodes Involved:**  
  - When Executed by Another Workflow

- **Node Details:**  
  - **When Executed by Another Workflow**  
    - Type: `Execute Workflow Trigger`  
    - Role: Starts the workflow with a predefined JSON example input.  
    - Configuration: Uses a JSON example input with keys `"name"` and `"audience"` (e.g., `{ "name": "Atlassian", "audience": "api.atlassian.com" }`).  
    - Inputs: None (trigger node)  
    - Outputs: Passes JSON input to the next node.  
    - Edge Cases: If input JSON is malformed or missing the `name` field, the AI prompt may fail or produce unreliable results. No explicit validation is present here.  
    - Version: 1.1  

#### 2.2 AI Processing

- **Overview:**  
  This block constructs a detailed prompt using the input name and sends it to the Wayfarer Large 70b Llama 3.3 model via OpenRouter. The AI agent attempts to identify OAuth2 settings and assign a confidence score to each extracted element.

- **Nodes Involved:**  
  - LLM Bus  
  - OpenRouter Chat Model

- **Node Details:**  
  - **LLM Bus**  
    - Type: `LangChain LLM Chain`  
    - Role: Prepares and sends a complex prompt to the AI model and receives the response.  
    - Configuration:  
      - Prompt instructs the AI to identify OAuth service name, audience, authorization URI, token URI, provide rationale, and a confidence score between 0.1 and 1.0.  
      - Includes examples for services like Jira (Atlassian), Sage, SAP, Google.  
      - Enforces output format and fallback improvisation with low confidence if data is unavailable.  
      - Uses expressions to inject the input JSON `name` into the prompt dynamically.  
    - Inputs: Receives JSON input from the trigger node.  
    - Outputs: Passes raw AI response to both the Structured Output Parser and the Code node.  
    - Edge Cases:  
      - AI may return malformed or unexpected output if prompt is ambiguous or input is invalid.  
      - Network or API errors with OpenRouter may cause retries or failure.  
      - Model temperature and penalties are set to balance creativity and accuracy.  
    - Version: 1.5  

  - **OpenRouter Chat Model**  
    - Type: `LangChain OpenRouter Chat Model`  
    - Role: Executes the AI inference using the specified Llama 3.3 model.  
    - Configuration:  
      - Model: `latitudegames/wayfarer-large-70b-llama-3.3`  
      - Parameters: topP=0.9, maxTokens=2500, temperature=0.5, presencePenalty=0.6, frequencyPenalty=0.5, maxRetries=2, responseFormat=json_object  
      - Credentials: Uses stored OpenRouter API key.  
    - Inputs: Receives prompt from LLM Bus node.  
    - Outputs: Returns AI-generated JSON object to LLM Bus.  
    - Edge Cases:  
      - API key invalid or rate limits exceeded.  
      - Model timeout or incomplete responses.  
    - Version: 1  

#### 2.3 Output Parsing

- **Overview:**  
  This block parses the AI model’s output using a JSON schema to extract the `action` and `text` fields, ensuring the data is in a consistent format for further processing.

- **Nodes Involved:**  
  - Structured Output Parser

- **Node Details:**  
  - **Structured Output Parser**  
    - Type: `LangChain Structured Output Parser`  
    - Role: Validates and parses the AI output against a manual JSON schema requiring `action` and `text` strings.  
    - Configuration:  
      - Schema requires two string properties: `action` and `text`.  
      - Ensures the AI output conforms to expected structure before passing downstream.  
    - Inputs: Receives AI output from LLM Bus.  
    - Outputs: Passes parsed JSON to the Code node for further transformation.  
    - Edge Cases:  
      - If AI output does not match schema, parsing fails and may cause workflow errors.  
      - No fallback or error handling configured.  
    - Version: 1.2  

#### 2.4 Output Conformation

- **Overview:**  
  This block uses a Code node to extract and normalize the AI output text into discrete fields: `service_name`, `audience`, `authorization_uri`, `token_uri`, `details`, and `confidence`. This structured format is suitable for downstream consumption or storage.

- **Nodes Involved:**  
  - Conform JSON

- **Node Details:**  
  - **Conform JSON**  
    - Type: `Code` (JavaScript)  
    - Role: Parses the multiline text output from the AI, splits it line-by-line, extracts key-value pairs, and converts the confidence score to a float.  
    - Configuration:  
      - Reads the `text` field from the AI output.  
      - Splits by newline and extracts values after colon-space delimiter.  
      - Returns a JSON object with clearly named properties.  
    - Inputs: Receives parsed AI output from Structured Output Parser.  
    - Outputs: Provides normalized JSON with OAuth2 settings and confidence.  
    - Edge Cases:  
      - If AI output text format changes (e.g., missing lines or different delimiters), parsing will fail or produce incorrect data.  
      - No error handling for malformed text.  
    - Version: 2  

---

### 3. Summary Table

| Node Name                     | Node Type                              | Functional Role                          | Input Node(s)                 | Output Node(s)              | Sticky Note                                                                                                  |
|-------------------------------|--------------------------------------|----------------------------------------|------------------------------|-----------------------------|--------------------------------------------------------------------------------------------------------------|
| When Executed by Another Workflow | Execute Workflow Trigger              | Entry point, receives input JSON       | None                         | LLM Bus                     | ## Start<br>**Trigger** input from the calling process.                                                     |
| LLM Bus                       | LangChain LLM Chain                   | Constructs prompt and sends to AI      | When Executed by Another Workflow, OpenRouter Chat Model, Structured Output Parser | Conform JSON                | ## AI Agent<br>**Prompt** input to find data.                                                              |
| OpenRouter Chat Model         | LangChain OpenRouter Chat Model       | Executes AI inference                   | LLM Bus                      | LLM Bus                     | ## OAuth2 Settings Finder with OpenRouter Chat Model and Llama 3.3<br>**Overview:** AI identifies OAuth URIs, token URI, audience. |
| Structured Output Parser      | LangChain Structured Output Parser    | Parses AI output into JSON structure   | LLM Bus                      | Conform JSON                | ## Output<br>**Parser** to grab the AI results into a JSON structure, according to the specified schema.    |
| Conform JSON                 | Code                                 | Normalizes AI output into structured JSON | Structured Output Parser      | None                        | ## Conform<br>**Output** to your process expectation.                                                       |
| Sticky Note                  | Sticky Note                          | Documentation                          | None                         | None                        | ## OAuth2 Settings Finder with OpenRouter Chat Model and Llama 3.3<br>**Overview:** AI agent identifies OAuth URIs, token URI, audience. |
| Sticky Note1                 | Sticky Note                          | Documentation                          | None                         | None                        | ## Start<br>**Trigger** input from the calling process.                                                     |
| Sticky Note2                 | Sticky Note                          | Documentation                          | None                         | None                        | ## AI Agent<br>**Prompt** input to find data.                                                              |
| Sticky Note3                 | Sticky Note                          | Documentation                          | None                         | None                        | ## Output<br>**Parser** to grab the AI results into a JSON structure, according to the specified schema.    |
| Sticky Note4                 | Sticky Note                          | Documentation                          | None                         | None                        | ## Conform<br>**Output** to your process expectation.                                                       |
| Sticky Note5                 | Sticky Note                          | Documentation                          | None                         | None                        | ## Purpose<br>This template assists users in obtaining OAuth2 settings using AI-powered insights. Ideal for developers and IT professionals. |
| Sticky Note6                 | Sticky Note                          | Documentation                          | None                         | None                        | ## Adaptability<br>Customize this template: update AI prompt, JSON schema, and Code logic as needed.        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node:**  
   - Add an **Execute Workflow Trigger** node named `When Executed by Another Workflow`.  
   - Configure it to use `jsonExample` input source with example JSON:  
     ```json
     {
       "name": "Atlassian",
       "audience": "api.atlassian.com"
     }
     ```  
   - This node acts as the entry point for the workflow.

2. **Add OpenRouter Chat Model Node:**  
   - Add a **LangChain OpenRouter Chat Model** node named `OpenRouter Chat Model`.  
   - Set the model to `latitudegames/wayfarer-large-70b-llama-3.3`.  
   - Configure parameters:  
     - `topP`: 0.9  
     - `maxTokens`: 2500  
     - `temperature`: 0.5  
     - `presencePenalty`: 0.6  
     - `frequencyPenalty`: 0.5  
     - `maxRetries`: 2  
     - `responseFormat`: `json_object`  
   - Set credentials to your OpenRouter API key.

3. **Add LLM Chain Node:**  
   - Add a **LangChain LLM Chain** node named `LLM Bus`.  
   - Configure the prompt with detailed instructions to identify OAuth2 settings from the input `name`.  
   - Include examples and specify output format with confidence scoring.  
   - Use expressions to dynamically inject `{{ $json.name }}` into the prompt.  
   - Connect the `When Executed by Another Workflow` node’s output to this node’s input.  
   - Connect this node’s AI language model input to the `OpenRouter Chat Model` node.  
   - Connect this node’s AI output parser input to the `Structured Output Parser` node (to be created next).

4. **Add Structured Output Parser Node:**  
   - Add a **LangChain Structured Output Parser** node named `Structured Output Parser`.  
   - Configure the schema manually as:  
     ```json
     {
       "$schema": "http://json-schema.org/draft-07/schema#",
       "title": "Generated schema for Root",
       "type": "object",
       "properties": {
         "action": { "type": "string" },
         "text": { "type": "string" }
       },
       "required": ["action", "text"]
     }
     ```  
   - Connect the AI output parser input to the `LLM Bus` node.

5. **Add Code Node to Normalize Output:**  
   - Add a **Code** node named `Conform JSON`.  
   - Paste the following JavaScript code to parse the AI output text:  
     ```javascript
     const items = $input.all();
     const originalText = items[0].json.output.text;
     const lines = originalText.split('\n');
     const service_name = lines[0].split(': ')[1];
     const audience = lines[1].split(': ')[1];
     const authorization_uri = lines[2].split(': ')[1];
     const token_uri = lines[3].split(': ')[1];
     const details = lines[4].split(': ')[1];
     const confidence = parseFloat(lines[5].split(': ')[1]);
     return [
       {
         json: {
           output: {
             service_name,
             audience,
             authorization_uri,
             token_uri,
             details,
             confidence
           }
         }
       }
     ];
     ```  
   - Connect the input of this node to the output of the `Structured Output Parser` node.

6. **Connect Workflow Nodes:**  
   - Connect `When Executed by Another Workflow` → `LLM Bus`  
   - Connect `LLM Bus` → `OpenRouter Chat Model` (AI language model input)  
   - Connect `LLM Bus` → `Structured Output Parser` (AI output parser input)  
   - Connect `Structured Output Parser` → `Conform JSON`

7. **Set Workflow Execution Order:**  
   - Ensure the execution order follows the connections above.

8. **Test the Workflow:**  
   - Execute the workflow with sample input (e.g., `"name": "Atlassian"`).  
   - Verify the output JSON contains correctly parsed OAuth2 settings and confidence score.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                          | Context or Link                                                                                         |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| This template is designed to assist users in obtaining OAuth2 settings using AI-powered insights. It is ideal for developers, IT professionals, or anyone working with APIs that require OAuth2 authentication. By leveraging the AI agent, users can simplify the process of extracting and validating key details such as the `authorization_url`, `token_url`, and `audience`. | Workflow Purpose (Sticky Note5)                                                                         |
| Confidence scoring is utilized to assess the trustworthiness of extracted data, with scores ranging from 0 < x ≤ 1 in increments of 0.01. This helps users gauge the reliability of the AI-generated OAuth2 settings.                                                                                                                                                  | Methodology (Sticky Note)                                                                               |
| Customize this template by updating the AI model prompt with details specific to your API or OAuth2 setup, adjusting the JSON schema in the Structured Output node, and modifying the Code node logic to suit your application's requirements.                                                                                                                         | Adaptability (Sticky Note6)                                                                             |
| Example JavaScript code snippet for the Code node is provided to parse and conform the AI output into structured JSON, which can be adapted for different output formats.                                                                                                                                                                                             | Setup Instructions (Sticky Note5)                                                                       |
| The AI prompt includes fallback instructions to always return a result, even if confidence is low, by improvising OAuth2 settings based on common patterns. This ensures the workflow never fails due to missing data but flags uncertain results with low confidence.                                                                                                   | AI Prompt Details (LLM Bus node parameters)                                                            |
| The workflow uses the Wayfarer Large 70b Llama 3.3 model via OpenRouter, which requires an API key credential. Ensure your OpenRouter account is properly configured in n8n credentials.                                                                                                                                                                              | Credential Requirement (OpenRouter Chat Model node)                                                    |

---

This documentation provides a detailed, structured, and comprehensive reference for understanding, reproducing, and modifying the OAuth2 Settings Finder workflow using n8n and AI integration.