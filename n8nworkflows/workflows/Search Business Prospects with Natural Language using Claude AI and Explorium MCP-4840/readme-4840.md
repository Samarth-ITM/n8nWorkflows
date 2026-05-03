Search Business Prospects with Natural Language using Claude AI and Explorium MCP

https://n8nworkflows.xyz/workflows/search-business-prospects-with-natural-language-using-claude-ai-and-explorium-mcp-4840


# Search Business Prospects with Natural Language using Claude AI and Explorium MCP

---

### 1. Workflow Overview

This workflow is designed to facilitate **searching business prospects using natural language queries** by leveraging the AI capabilities of Claude (Anthropic) and Explorium MCP (Market Contact Platform). The main use case is to allow users to input natural language requests for B2B prospect data, which the workflow converts into structured API calls to the Explorium MCP, retrieves matching prospect data, processes it, and outputs it in a user-friendly file format.

The workflow is logically divided into the following functional blocks:

- **1.1 Input Reception & Query Handling:** Receives chat messages as input, handles new queries or validation refinements.
- **1.2 AI Processing & Request Generation:** Uses Claude AI to interpret natural language, generate structured JSON requests compliant with Explorium MCP API specifications, and parse AI outputs.
- **1.3 Validation & Refinement Loop:** Validates AI-generated API request JSON, prompts for corrections if necessary, and loops back for regeneration.
- **1.4 Explorium MCP API Interaction:** Sends validated requests to the Explorium MCP API, handles paginated responses, and merges all prospect data.
- **1.5 Data Preparation & Export:** Processes raw data into CSV-friendly structure and converts it into a file for download or further use.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Input Reception & Query Handling

**Overview:**  
This block receives natural language chat messages from users and manages the input for initial queries or corrected outputs following validation errors.

**Nodes Involved:**  
- When chat message received  
- Chat or Refinement  

**Node Details:**

- **When chat message received**  
  - Type: Chat Trigger (LangChain)  
  - Role: Entry point, listens for incoming chat messages via webhook.  
  - Configuration: Default webhook with unique ID. No special parameters.  
  - Input: External chat message via webhook.  
  - Output: Emits JSON with `chatInput` and `sessionId`.  
  - Edge Cases: Webhook connectivity issues, malformed chat input.

- **Chat or Refinement**  
  - Type: Code  
  - Role: Combines either the user's original chat input or error-corrected input to feed the AI agent.  
  - Configuration: JavaScript concatenates `$json.chatInput` or `$json.errorInput` into `combinedInput`.  
  - Input: JSON from chat trigger or validation prompter.  
  - Output: JSON with `combinedInput` field for AI agent use.  
  - Edge Cases: Missing `errorInput` or `chatInput` could cause empty combined input.

---

#### 2.2 AI Processing & Request Generation

**Overview:**  
Interprets the natural language input using Claude AI to generate structured JSON requests formatted specifically for the Explorium MCP API. It includes output parsing and integration with memory for context retention.

**Nodes Involved:**  
- Anthropic Chat Model  
- Simple Memory  
- AI Agent  
- Output Parser  

**Node Details:**

- **Anthropic Chat Model**  
  - Type: LangChain AI Language Model Node (Claude Sonnet 4)  
  - Role: Provides AI-generated responses based on the combined input.  
  - Configuration: Model set to `claude-sonnet-4-20250514`, thinking disabled.  
  - Credentials: Anthropic API key required.  
  - Input: `combinedInput` from Chat or Refinement node.  
  - Output: Raw AI textual response.  
  - Edge Cases: API quota limits, network timeouts, authentication errors.

- **Simple Memory**  
  - Type: LangChain Memory Buffer Window  
  - Role: Maintains a sliding window of conversation context (length 100) for better AI understanding.  
  - Configuration: Context window length set to 100.  
  - Input: Connected to AI Agent node for context storage.  
  - Output: Contextual data feeding back to AI Agent.  
  - Edge Cases: Context overflow or memory leaks if not managed.

- **AI Agent**  
  - Type: LangChain Agent Node  
  - Role: Core logic node that uses the AI model to generate structured JSON for Explorium MCP API requests.  
  - Configuration:  
    - Input text: `combinedInput` expression.  
    - System message: Detailed prompt instructing the AI on JSON structure, filter validation, error handling, and formatting rules.  
    - Output parser enabled to enforce JSON format.  
    - Retry on fail enabled to handle transient AI errors.  
  - Input: Receives combined input and memory context, invokes Anthropic Chat Model.  
  - Output: AI-generated JSON request structure.  
  - Edge Cases: AI misinterpretation, formatting errors, exceeding iteration limits.

- **Output Parser**  
  - Type: LangChain Output Parser Structured  
  - Role: Parses AI textual output strictly into JSON using a provided schema example for validation.  
  - Configuration: JSON schema example matching MCP API request format with filters.  
  - Input: AI Agent raw output.  
  - Output: Parsed JSON object for further validation.  
  - Edge Cases: Parsing failures if AI output deviates from schema or is malformed.

---

#### 2.3 Validation & Refinement Loop

**Overview:**  
Validates the AI-generated API request JSON against MCP filters and allowed values. If invalid, it generates a detailed prompt to instruct the AI to correct the response and loops back for regeneration.

**Nodes Involved:**  
- API Call Validation  
- Is API Call Valid? (If)  
- Validation Prompter  

**Node Details:**

- **API Call Validation**  
  - Type: Code  
  - Role: Performs detailed validation of the AI-generated JSON filters against allowed keys, value types, allowed enums, ranges, and duplicates.  
  - Configuration: Custom JavaScript with hardcoded valid lists for company sizes, ages, revenues, job levels, departments, and regex for country codes.  
  - Input: Parsed JSON from Output Parser.  
  - Output: Augmented JSON with `isValid` boolean and `validationErrors` array.  
  - Edge Cases: Unexpected filter keys, invalid value types, empty arrays, invalid ranges, duplicate values.

- **Is API Call Valid?**  
  - Type: If  
  - Role: Branches workflow based on `isValid` flag.  
  - Configuration: Condition checks if `isValid === true`.  
  - Input: Output of API Call Validation.  
  - Output:  
    - True branch: Proceed with API call.  
    - False branch: Trigger Validation Prompter.  
  - Edge Cases: Unexpected missing `isValid` field.

- **Validation Prompter**  
  - Type: Code  
  - Role: Formats a detailed error message including user query, AI output, and validation errors to send back to the AI for correction.  
  - Configuration: Builds a multi-line string prompt with embedded JSON and error list.  
  - Input: Original user query, AI output, and validation errors.  
  - Output: JSON containing a message instructing regeneration.  
  - Edge Cases: Empty or malformed error lists, missing fields.

---

#### 2.4 Explorium MCP API Interaction

**Overview:**  
Sends validated structured requests to the Explorium MCP API endpoint, supports automatic pagination, collects all pages, and merges prospect data arrays into a single dataset.

**Nodes Involved:**  
- Explorium Prospects API Call  
- Merge All Pages  
- Extract "data"  

**Node Details:**

- **Explorium Prospects API Call**  
  - Type: HTTP Request  
  - Role: Sends POST requests to Explorium MCP prospects endpoint with JSON body from validated AI output.  
  - Configuration:  
    - URL: `https://api.explorium.ai/v1/prospects`  
    - Method: POST  
    - Pagination: Auto-pagination on `page` parameter in body; stops when response data array is empty.  
    - Headers: Accept JSON, uses HTTP header authentication with stored credentials.  
  - Input: Validated JSON request from Is API Call Valid? node.  
  - Output: Responses with prospect data in paginated format.  
  - Edge Cases: API rate limits, invalid credentials, network failures, empty results.

- **Merge All Pages**  
  - Type: Code  
  - Role: Merges multiple paginated response items into a single unified array under `data` key.  
  - Configuration: JavaScript aggregates all `data` arrays from input items into one.  
  - Input: All paginated response items from Explorium Prospects API Call.  
  - Output: Single JSON with combined `data` array.  
  - Edge Cases: Missing or malformed `data` arrays in input.

- **Extract "data"**  
  - Type: Split Out  
  - Role: Extracts each individual prospect object from the combined `data` array, creating separate output items.  
  - Configuration: Field to split out set to `"data"`.  
  - Input: Single item with combined `data` array.  
  - Output: Multiple items, one per prospect.  
  - Edge Cases: Empty `data` arrays produce no output items.

---

#### 2.5 Data Preparation & Export

**Overview:**  
Transforms the raw prospect data into a flattened, CSV-compatible format and converts it into a downloadable file.

**Nodes Involved:**  
- Prepare for CSV  
- Convert to File  

**Node Details:**

- **Prepare for CSV**  
  - Type: Code  
  - Role: Maps and flattens prospect fields into a simplified structure, concatenating arrays into strings for CSV compatibility.  
  - Configuration:  
    - Extracts fields such as prospect_id, names, location, LinkedIn, experience, skills, company info, job title/level, business_id.  
    - Arrays (experience, skills, interests, job seniority) are joined with appropriate separators.  
    - Null defaults for missing fields.  
  - Input: Individual prospect items from Extract "data".  
  - Output: Structured items ready for CSV export.  
  - Edge Cases: Missing fields, non-array fields for expected arrays.

- **Convert to File**  
  - Type: Convert to File  
  - Role: Converts structured data into a file format (default CSV or as configured) for download or further use.  
  - Configuration: Default options.  
  - Input: Items prepared for CSV.  
  - Output: File object containing CSV data.  
  - Edge Cases: Large datasets could cause memory issues.

---

### 3. Summary Table

| Node Name                 | Node Type                          | Functional Role                                       | Input Node(s)                  | Output Node(s)                     | Sticky Note                              |
|---------------------------|----------------------------------|-----------------------------------------------------|--------------------------------|-----------------------------------|-----------------------------------------|
| When chat message received| Chat Trigger (LangChain)          | Entry point, receives user chat input               | -                              | Chat or Refinement                |                                         |
| Chat or Refinement        | Code                             | Combines chat input or validation error for AI      | When chat message received, Validation Prompter | AI Agent                     |                                         |
| Anthropic Chat Model      | LangChain AI Model (Claude)      | Generates AI response based on combined input       | AI Agent (languageModel)        | AI Agent                         | Requires Anthropic API credentials       |
| Simple Memory             | LangChain Memory Buffer Window   | Retains conversation context for AI                  | AI Agent (memory)               | AI Agent                         |                                         |
| AI Agent                  | LangChain Agent                  | Generates structured JSON for Explorium MCP          | Chat or Refinement, Anthropic Chat Model, Simple Memory | API Call Validation          | Core logic node with detailed system prompt |
| Output Parser             | LangChain Output Parser           | Parses AI textual output into validated JSON          | AI Agent                       | API Call Validation              |                                         |
| API Call Validation       | Code                             | Validates AI-generated JSON filters and values        | Output Parser                  | Is API Call Valid?               |                                         |
| Is API Call Valid?        | If                               | Branches on validation result                         | API Call Validation            | Explorium Prospects API Call (true), Validation Prompter (false) |                                         |
| Validation Prompter       | Code                             | Formats validation errors to prompt AI correction     | API Call Validation            | Chat or Refinement              |                                         |
| Explorium Prospects API Call | HTTP Request                    | Sends API request to Explorium MCP, handles pagination | Is API Call Valid? (true)      | Merge All Pages                 | Requires HTTP header auth credentials    |
| Merge All Pages           | Code                             | Merges paginated results into single dataset          | Explorium Prospects API Call   | Extract "data"                  |                                         |
| Extract "data"            | Split Out                        | Splits combined data array into individual items      | Merge All Pages                | Prepare for CSV                 |                                         |
| Prepare for CSV           | Code                             | Flattens and formats prospect data for CSV export    | Extract "data"                 | Convert to File                 |                                         |
| Convert to File           | Convert To File                  | Converts data to downloadable file                     | Prepare for CSV                | -                              |                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "When chat message received" node**  
   - Type: LangChain Chat Trigger  
   - Configure webhook (default) to receive chat input. No parameters needed.  

2. **Create "Chat or Refinement" Code node**  
   - Connect input from "When chat message received".  
   - JavaScript code: output `combinedInput` as either `errorInput` or `chatInput` from incoming JSON.  
   - Output feeds AI Agent.  

3. **Create "Anthropic Chat Model" node**  
   - Type: LangChain AI Language Model (choose Anthropic model "claude-sonnet-4-20250514").  
   - Disable thinking mode.  
   - Set credentials with Anthropic API key.  
   - Connect as AI language model input to AI Agent.  

4. **Create "Simple Memory" node**  
   - Type: LangChain Memory Buffer Window.  
   - Set context window length: 100.  
   - Connect memory input to AI Agent node.  

5. **Create "AI Agent" node**  
   - Type: LangChain Agent.  
   - Set input text: expression `={{ $json.combinedInput }}`.  
   - Paste system message prompt instructing AI to generate Explorium MCP JSON request with filter rules and error handling (as detailed in the original prompt).  
   - Enable output parser and retry on fail.  
   - Connect inputs from "Chat or Refinement", "Anthropic Chat Model", and "Simple Memory".  
   - Connect output to "Output Parser" node.  

6. **Create "Output Parser" node**  
   - Type: LangChain Output Parser Structured.  
   - Paste JSON schema example enforcing required MCP request structure and filters.  
   - Connect input from "AI Agent".  
   - Connect output to "API Call Validation".  

7. **Create "API Call Validation" Code node**  
   - Paste JavaScript code that validates keys, values, duplicates, and range constraints for MCP filters.  
   - Connect input from "Output Parser".  
   - Connect output to "Is API Call Valid?" node.  

8. **Create "Is API Call Valid?" node**  
   - Type: If  
   - Condition: `$json.isValid === true` (strict boolean).  
   - True branch to "Explorium Prospects API Call".  
   - False branch to "Validation Prompter".  

9. **Create "Validation Prompter" Code node**  
   - Paste JavaScript code formatting detailed error message with user query, AI output, and validation errors.  
   - Connect input from "API Call Validation".  
   - Connect output back to "Chat or Refinement" (loop for correction).  

10. **Create "Explorium Prospects API Call" HTTP Request node**  
    - Method: POST  
    - URL: `https://api.explorium.ai/v1/prospects`  
    - Body type: JSON, body expression `={{ $json.output }}` (validated request).  
    - Headers: Accept: application/json.  
    - Authentication: HTTP Header Auth with valid Explorium API key.  
    - Enable pagination on `page` parameter in request body; stop when returned data array is empty.  
    - Connect input from true branch of "Is API Call Valid?".  
    - Connect output to "Merge All Pages".  

11. **Create "Merge All Pages" Code node**  
    - JavaScript to concatenate all `data` arrays from paginated responses into a single combined array under key `data`.  
    - Connect input from "Explorium Prospects API Call".  
    - Connect output to "Extract \"data\"".  

12. **Create "Extract \"data\"" node**  
    - Type: Split Out  
    - Field to split out: `data`  
    - Connect input from "Merge All Pages".  
    - Connect output to "Prepare for CSV".  

13. **Create "Prepare for CSV" Code node**  
    - JavaScript to map and flatten each prospect’s fields into CSV-compatible format (join arrays, handle nulls).  
    - Connect input from "Extract \"data\"".  
    - Connect output to "Convert to File".  

14. **Create "Convert to File" node**  
    - Type: Convert to File  
    - Default options (e.g., CSV).  
    - Connect input from "Prepare for CSV".  
    - Output provides downloadable file.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                           | Context or Link                                                                                                                  |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------|
| This workflow is designed to strictly comply with Explorium MCP API filter specifications and validates AI outputs accordingly.                                      | Key for ensuring API calls do not fail due to invalid parameters.                                                               |
| The AI system prompt includes detailed instructions for strict JSON output formatting, validation, and error correction cycle.                                       | Ensures consistent API requests and user feedback on errors.                                                                    |
| Uses Claude Sonnet 4 AI model from Anthropic for natural language understanding and JSON generation.                                                                 | Requires Anthropic API credentials configured in n8n.                                                                           |
| Pagination handling in the API request node automatically fetches all pages until no more data is returned.                                                          | Ensures complete data retrieval without manual intervention.                                                                     |
| The final output is a CSV-formatted file consolidating all relevant prospect information for easy export or downstream processing.                                    | Useful for direct integration with CRM or marketing automation tools.                                                           |
| For further information on Explorium MCP API and its filter parameters, consult Explorium documentation and official API references.                                | https://explorium.ai/docs/api                                                                                                    |

---

**Disclaimer:** The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.

---