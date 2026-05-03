Generate AI Prompts with Google Gemini and store them in Airtable

https://n8nworkflows.xyz/workflows/generate-ai-prompts-with-google-gemini-and-store-them-in-airtable-3027


# Generate AI Prompts with Google Gemini and store them in Airtable

### 1. Workflow Overview

This workflow automates the generation of AI prompts using Google Gemini (PaLM) and stores the generated prompts in Airtable for centralized management. It is designed for users who want to create, categorize, and archive AI prompts efficiently, enabling rapid prototyping and reuse in AI agent applications.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Receives chat messages as trigger inputs.
- **1.2 Prompt Generation:** Uses Google Gemini to generate a structured AI prompt based on the input.
- **1.3 Prompt Refinement and Parsing:** Parses and auto-fixes the generated prompt output to ensure structure and correctness.
- **1.4 Prompt Categorization:** Categorizes and names the generated prompt using AI.
- **1.5 Data Preparation:** Sets and formats the prompt fields for storage.
- **1.6 Data Storage:** Stores the prompt in Airtable.
- **1.7 Output Delivery:** Returns confirmation and the generated prompt back to the chat interface.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block listens for incoming chat messages to trigger the workflow.

- **Nodes Involved:**  
  - When chat message received

- **Node Details:**  
  - **When chat message received**  
    - Type: Chat Trigger (LangChain)  
    - Role: Entry point; triggers workflow on receiving a chat message.  
    - Configuration: Default webhook with no additional options.  
    - Inputs: External chat message via webhook.  
    - Outputs: Passes chat message data downstream.  
    - Edge Cases: Webhook misconfiguration, missing or malformed chat messages.  
    - Version: 1.1

#### 2.2 Prompt Generation

- **Overview:**  
  Generates a new AI prompt based on the received chat message, using a detailed prompt engineering template to ensure high-quality, structured output.

- **Nodes Involved:**  
  - Google Gemini Chat Model  
  - Generate a new prompt  
  - Edit Fields

- **Node Details:**  
  - **Google Gemini Chat Model**  
    - Type: Language Model (Google Gemini)  
    - Role: Runs the Google Gemini model to process input text.  
    - Configuration: Model set to "models/gemini-2.0-flash-lite", topP=1 for diversity control.  
    - Credentials: Uses Google Gemini (PaLM) API credentials.  
    - Inputs: Receives chat message from trigger node.  
    - Outputs: Passes AI-generated text to "Generate a new prompt".  
    - Edge Cases: API key invalid, rate limits, network timeouts.  
    - Version: 1

  - **Generate a new prompt**  
    - Type: Chain LLM (LangChain)  
    - Role: Constructs a detailed prompt engineering template instructing the AI to create optimized prompts for n8n AI agents.  
    - Configuration: Contains a comprehensive multi-section prompt template with instructions, examples, and rules for AI prompt generation.  
    - Inputs: Receives AI output from Google Gemini Chat Model.  
    - Outputs: Passes structured prompt text to "Edit Fields".  
    - Key Expressions: Uses `={{ $json.text }}` to pass text dynamically.  
    - Edge Cases: Template errors, incomplete AI responses.  
    - Version: 1.5

  - **Edit Fields**  
    - Type: Set Node  
    - Role: Extracts and sets the "text" field from the AI output for further processing.  
    - Configuration: Assigns `text` field from incoming JSON.  
    - Inputs: Receives prompt text from "Generate a new prompt".  
    - Outputs: Passes cleaned text to "Categorize and name Prompt".  
    - Edge Cases: Missing or malformed text field.  
    - Version: 3.4

#### 2.3 Prompt Refinement and Parsing

- **Overview:**  
  Parses the AI-generated prompt output to ensure it matches a defined JSON schema and auto-fixes any structural issues.

- **Nodes Involved:**  
  - Structured Output Parser  
  - Auto-fixing Output Parser

- **Node Details:**  
  - **Structured Output Parser**  
    - Type: Output Parser (LangChain)  
    - Role: Parses AI output into a structured JSON format with fields "name" and "category".  
    - Configuration: Uses a JSON schema example specifying required fields.  
    - Inputs: Receives AI-generated text.  
    - Outputs: Passes parsed JSON to "Auto-fixing Output Parser".  
    - Edge Cases: Parsing failures if AI output deviates from schema.  
    - Version: 1.2

  - **Auto-fixing Output Parser**  
    - Type: Output Parser (LangChain)  
    - Role: Automatically corrects parsing errors or malformed JSON output from the previous parser.  
    - Configuration: Default options.  
    - Inputs: Receives parsed JSON from "Structured Output Parser".  
    - Outputs: Passes corrected JSON to "Categorize and name Prompt".  
    - Edge Cases: Unfixable parsing errors, API failures.  
    - Version: 1

#### 2.4 Prompt Categorization

- **Overview:**  
  Uses AI to categorize the prompt and assign a meaningful name, enhancing organization and retrieval.

- **Nodes Involved:**  
  - Categorize and name Prompt  
  - Google Gemini Chat Model1

- **Node Details:**  
  - **Google Gemini Chat Model1**  
    - Type: Language Model (Google Gemini)  
    - Role: Runs Google Gemini model to assist categorization.  
    - Configuration: Same model as before, "models/gemini-2.0-flash-lite", topP=1.  
    - Credentials: Google Gemini (PaLM) API.  
    - Inputs: Receives parsed prompt JSON.  
    - Outputs: Passes AI output to "Categorize and name Prompt" and "Auto-fixing Output Parser" (parallel).  
    - Edge Cases: API errors, rate limits.  
    - Version: 1

  - **Categorize and name Prompt**  
    - Type: Chain LLM (LangChain)  
    - Role: Takes prompt text and categorizes it into a relevant category with a name.  
    - Configuration: Uses input text and a message instructing categorization.  
    - Inputs: Receives AI-generated prompt text and parsed JSON.  
    - Outputs: Passes categorized output to "set prompt fields".  
    - Edge Cases: Misclassification, incomplete output.  
    - Version: 1.5

#### 2.5 Data Preparation

- **Overview:**  
  Prepares the final data fields for storage by setting the prompt name, category, and prompt text.

- **Nodes Involved:**  
  - set prompt fields

- **Node Details:**  
  - **set prompt fields**  
    - Type: Set Node  
    - Role: Maps AI output fields "name" and "category" and the prompt text into fields for Airtable.  
    - Configuration: Assigns "name" and "category" from AI output; "Prompt" from "Edit Fields" node's text.  
    - Inputs: Receives categorized prompt JSON.  
    - Outputs: Passes data to "add to airtable".  
    - Edge Cases: Missing fields, data type mismatches.  
    - Version: 3.4

#### 2.6 Data Storage

- **Overview:**  
  Creates a new record in Airtable with the generated prompt data.

- **Nodes Involved:**  
  - add to airtable

- **Node Details:**  
  - **add to airtable**  
    - Type: Airtable Node  
    - Role: Inserts a new record into the specified Airtable base and table.  
    - Configuration:  
      - Base: "Prompt Library" (app994hU3fOw0ssrx)  
      - Table: "Prompt Library" table (tbldwJrCK2HmAeknA)  
      - Columns mapped: Name, Prompt, Category  
    - Credentials: Airtable Personal Access Token account.  
    - Inputs: Receives prompt data from "set prompt fields".  
    - Outputs: Passes success confirmation to "Return results".  
    - Edge Cases: Invalid credentials, incorrect base/table IDs, column mismatches, API rate limits.  
    - Version: 2.1

#### 2.7 Output Delivery

- **Overview:**  
  Sends back the generated prompt text as confirmation to the chat interface.

- **Nodes Involved:**  
  - Return results

- **Node Details:**  
  - **Return results**  
    - Type: Set Node  
    - Role: Sets the output text to be returned to the chat interface.  
    - Configuration: Sets "text" field to the prompt text from "Generate a new prompt" node.  
    - Inputs: Receives data from "add to airtable".  
    - Outputs: Returns final response to the chat trigger.  
    - Edge Cases: Missing prompt text, output formatting issues.  
    - Version: 3.4

---

### 3. Summary Table

| Node Name                 | Node Type                         | Functional Role                      | Input Node(s)                 | Output Node(s)               | Sticky Note                                                                                  |
|---------------------------|----------------------------------|------------------------------------|------------------------------|-----------------------------|----------------------------------------------------------------------------------------------|
| When chat message received | Chat Trigger (LangChain)          | Entry trigger for chat messages    | -                            | Generate a new prompt        |                                                                                              |
| Google Gemini Chat Model   | Language Model (Google Gemini)    | Runs Gemini model for prompt gen   | When chat message received    | Generate a new prompt        |                                                                                              |
| Generate a new prompt      | Chain LLM (LangChain)             | Creates detailed AI prompt template| Google Gemini Chat Model      | Edit Fields                 |                                                                                              |
| Edit Fields               | Set Node                         | Extracts prompt text field          | Generate a new prompt         | Categorize and name Prompt   |                                                                                              |
| Structured Output Parser   | Output Parser (LangChain)         | Parses AI output into JSON          | Google Gemini Chat Model1     | Auto-fixing Output Parser    |                                                                                              |
| Auto-fixing Output Parser  | Output Parser (LangChain)         | Auto-corrects parsing errors        | Structured Output Parser      | Categorize and name Prompt   |                                                                                              |
| Google Gemini Chat Model1  | Language Model (Google Gemini)    | Runs Gemini model for categorization| Auto-fixing Output Parser     | Categorize and name Prompt, Auto-fixing Output Parser |                                                                                              |
| Categorize and name Prompt | Chain LLM (LangChain)             | Categorizes and names prompt        | Edit Fields, Google Gemini Chat Model1, Auto-fixing Output Parser | set prompt fields           |                                                                                              |
| set prompt fields          | Set Node                         | Prepares data fields for Airtable  | Categorize and name Prompt    | add to airtable             |                                                                                              |
| add to airtable            | Airtable Node                    | Stores prompt in Airtable           | set prompt fields            | Return results              | Map Airtable base and table correctly; use Personal Access Token credentials                  |
| Return results             | Set Node                         | Sends confirmation back to chat    | add to airtable              | -                           |                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node**  
   - Add **When chat message received** (LangChain Chat Trigger) node.  
   - Configure webhook with default settings. This node will start the workflow on incoming chat messages.

2. **Add Google Gemini Chat Model Node**  
   - Add **Google Gemini Chat Model** node.  
   - Set modelName to `"models/gemini-2.0-flash-lite"`.  
   - Set `topP` to `1`.  
   - Assign Google Gemini (PaLM) API credentials.  
   - Connect input from **When chat message received** node.

3. **Add Chain LLM Node for Prompt Generation**  
   - Add **Generate a new prompt** node (Chain LLM).  
   - Paste the detailed prompt engineering template into the messages parameter.  
   - Connect input from **Google Gemini Chat Model** node.

4. **Add Set Node to Extract Text**  
   - Add **Edit Fields** node (Set).  
   - Configure to assign a new field `text` with value `={{ $json.text }}`.  
   - Connect input from **Generate a new prompt** node.

5. **Add Google Gemini Chat Model1 Node for Categorization**  
   - Add another **Google Gemini Chat Model** node (named Google Gemini Chat Model1).  
   - Use same model and credentials as before.  
   - Connect input from **Auto-fixing Output Parser** node (to be added next).

6. **Add Structured Output Parser Node**  
   - Add **Structured Output Parser** node.  
   - Configure JSON schema example:  
     ```json
     {
       "name": "Name of the prompt",
       "category": "the prompt category"
     }
     ```  
   - Connect input from **Google Gemini Chat Model1** node.

7. **Add Auto-fixing Output Parser Node**  
   - Add **Auto-fixing Output Parser** node.  
   - Connect input from **Structured Output Parser** node.

8. **Add Chain LLM Node for Categorization and Naming**  
   - Add **Categorize and name Prompt** node (Chain LLM).  
   - Configure to take `={{ $json.text }}` as input and message `"Categorize the above prompt into a category that it can fall into"`.  
   - Connect inputs from **Edit Fields** node and **Google Gemini Chat Model1** node (parallel inputs).  
   - Also connect input from **Auto-fixing Output Parser** node.

9. **Add Set Node to Prepare Prompt Fields**  
   - Add **set prompt fields** node (Set).  
   - Assign fields:  
     - `name` = `={{ $json.output.name }}`  
     - `category` = `={{ $json.output.category }}`  
     - `Prompt` = `={{ $('Edit Fields').item.json.text }}`  
   - Connect input from **Categorize and name Prompt** node.

10. **Add Airtable Node to Store Prompt**  
    - Add **add to airtable** node (Airtable).  
    - Configure:  
      - Base: Select or enter your Airtable base (e.g., "Prompt Library").  
      - Table: Select or enter your Airtable table (e.g., "Prompt Library").  
      - Map columns:  
        - Name → `={{ $json.name }}`  
        - Prompt → `={{ $json.Prompt }}`  
        - Category → `={{ $json.category }}`  
    - Assign Airtable Personal Access Token credentials.  
    - Connect input from **set prompt fields** node.

11. **Add Set Node to Return Results**  
    - Add **Return results** node (Set).  
    - Assign field `text` = `={{ $('Generate a new prompt').item.json.text }}`.  
    - Connect input from **add to airtable** node.

12. **Connect Workflow**  
    - Connect nodes in this order:  
      `When chat message received` → `Google Gemini Chat Model` → `Generate a new prompt` → `Edit Fields` → `Categorize and name Prompt` → `set prompt fields` → `add to airtable` → `Return results`.  
    - Connect `Google Gemini Chat Model1` node input from `Auto-fixing Output Parser`.  
    - Connect `Structured Output Parser` from `Google Gemini Chat Model1`.  
    - Connect `Auto-fixing Output Parser` from `Structured Output Parser`.  
    - Connect `Categorize and name Prompt` from `Edit Fields`, `Google Gemini Chat Model1`, and `Auto-fixing Output Parser`.

13. **Activate Workflow**  
    - Save and activate the workflow.  
    - Test by sending a chat message to the webhook URL.

---

### 5. General Notes & Resources

| Note Content                                                                                                         | Context or Link                                                                                  |
|----------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Clone the [Prompt Library Airtable base](https://airtable.com/app994hU3fOw0ssrx/shrRxcst3vzWMKFcR) before use.       | Airtable base for storing generated prompts                                                    |
| Follow the creator on LinkedIn for more workflows and tips: [Zacharia Kimotho](https://www.linkedin.com/in/zacharia-kimotho/) | Author and workflow source                                                                     |
| Ensure Google Gemini API is enabled and credentials are valid to avoid 400 Bad Request errors.                       | Troubleshooting API authentication issues                                                     |
| Airtable Personal Access Token must have correct permissions for the base and table used.                            | Prevents Airtable record creation failures                                                    |
| Customize the prompt template in "Generate a new prompt" node to fit your specific AI agent use case.                | Allows tailoring prompt generation to business needs                                          |
| Monitor workflow execution in n8n editor to debug data flow and errors.                                              | Essential for troubleshooting and ensuring correct operation                                  |

---

This document provides a complete, detailed reference to understand, reproduce, and maintain the "Generate AI Prompts with Google Gemini and store them in Airtable" workflow. It covers all nodes, their configurations, connections, and potential failure points, enabling both advanced users and automation agents to work effectively with this workflow.