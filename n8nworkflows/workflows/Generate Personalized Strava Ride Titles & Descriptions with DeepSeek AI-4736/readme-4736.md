Generate Personalized Strava Ride Titles & Descriptions with DeepSeek AI

https://n8nworkflows.xyz/workflows/generate-personalized-strava-ride-titles---descriptions-with-deepseek-ai-4736


# Generate Personalized Strava Ride Titles & Descriptions with DeepSeek AI

### 1. Workflow Overview

This n8n workflow automates the process of generating personalized titles and descriptions for Strava cycling activities using AI, specifically the DeepSeek AI model accessed via OpenRouter. It targets cyclists who want to enhance their ride posts on Strava with engaging and contextually relevant content, improving social interaction and motivation.

The workflow logically divides into the following blocks:

- **1.1 Input Reception:** Triggered by creation of a new Strava activity, retrieving raw activity data.
- **1.2 Data Enrichment and Transformation:** Adds custom fields and flattens the complex JSON data into a readable text format.
- **1.3 AI Processing:** Sends the formatted activity data to an AI agent that generates a catchy title and detailed description.
- **1.4 Post-Processing and Extraction:** Parses the AI output into structured fields for title and description.
- **1.5 Strava Update:** Automatically updates the original Strava activity with the AI-generated title and description.

Supporting sticky notes provide detailed explanations and instructions for each step.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** Listens for new Strava activities and triggers the workflow with the raw activity data.
- **Nodes Involved:**  
  - *Strava Trigger*
- **Node Details:**  
  - **Strava Trigger**  
    - Type: Strava Trigger node  
    - Role: Entry point; monitors creation of new activities on connected Strava account.  
    - Configuration:  
      - Event: `create`  
      - Object: `activity`  
      - Webhook enabled with a specific webhook ID for real-time triggers.  
    - Credentials: Strava OAuth2 with required scopes (`activity:read_all`, `activity:write`).  
    - Input Connections: None (trigger node).  
    - Output Connections: To "Code" node for data enrichment.  
    - Possible Failures: OAuth token expiration, Strava API rate limits, webhook registration errors.  
    - Sticky Note: Explains OAuth2 credential setup and trigger role.

#### 2.2 Data Enrichment and Transformation

- **Overview:** Prepares and formats the raw activity data for AI consumption by adding custom fields and converting nested JSON into a flat, readable string.
- **Nodes Involved:**  
  - *Code*  
  - *Combine Everything*  
- **Node Details:**  
  - **Code (Data Enrichment)**  
    - Type: Code (JavaScript) node  
    - Role: Adds or modifies fields in the input JSON; currently adds a dummy field `myNewField = 1` for demonstration or future use.  
    - Configuration: Simple loop over input items to add the field.  
    - Input: From Strava Trigger  
    - Output: To "Combine Everything" node  
    - Failures: JavaScript runtime errors, unexpected input structure.  
  - **Combine Everything (Data Flattening)**  
    - Type: Code (JavaScript) node  
    - Role: Recursively flattens the complex JSON data into a single string with key-value pairs separated by newlines, making it AI-readable.  
    - Configuration: Uses a recursive function to flatten nested objects, concatenates all items with separators.  
    - Input: From "Code" node  
    - Output: To "Strava Social Manager" AI node  
    - Failures: Deeply nested or circular JSON could cause issues; large payloads may impact performance.  
    - Sticky Note: Explains the purpose and mechanics of data flattening.

#### 2.3 AI Processing

- **Overview:** Sends the formatted activity data to DeepSeek AI via OpenRouter to generate a creative, personalized title and description for the ride.
- **Nodes Involved:**  
  - *Strava Social Manager*  
  - *OpenRouter Chat Model* (used internally by the agent node)  
- **Node Details:**  
  - **Strava Social Manager**  
    - Type: Langchain Agent node  
    - Role: Defines the AI prompt, sends data to AI, and receives generated text output.  
    - Configuration:  
      - Prompt: Detailed instructions for AI to craft a catchy title (max 10 words) and an inspiring, personalized description (3-5 sentences) based on ride data.  
      - Data passed via variable substitution (`{{ $json.data }}`) containing flattened activity details.  
      - Agent Type: `conversationalAgent`  
    - Input: From "Combine Everything"  
    - Output: To "Edit Fields" node  
    - Credentials: OpenRouter API configured with DeepSeek model (`deepseek-r1:free`).  
    - Failures: API key invalid, rate limiting, network errors, malformed prompt or data.  
    - Sticky Note: Describes the AI prompt and expected output format.

  - **OpenRouter Chat Model**  
    - Type: Langchain language model node  
    - Role: Connects the agent node to the OpenRouter DeepSeek AI model.  
    - Configuration: Model name specified; no additional options.  
    - Input: From "Strava Social Manager" (agent) node.  
    - Output: To "Strava Social Manager" node.  
    - Failures: Authentication errors, API downtime.

#### 2.4 Post-Processing and Extraction

- **Overview:** Parses the AI-generated text output to extract the title and description into dedicated fields to prepare for updating Strava.
- **Nodes Involved:**  
  - *Edit Fields*  
- **Node Details:**  
  - **Edit Fields**  
    - Type: Set node  
    - Role: Splits the multiline AI response string into separate `Name` (title) and `Description` fields using JavaScript expressions.  
    - Configuration:  
      - Extracts title from the first line after `Titre :` prefix.  
      - Extracts description from the second line after `Description :` prefix.  
    - Input: From "Strava Social Manager"  
    - Output: To "Strava" node  
    - Failures: Unexpected AI output format, missing delimiters, empty strings.  
    - Sticky Note: Explains the extraction logic.

#### 2.5 Strava Update

- **Overview:** Updates the original Strava activity’s title and description fields with the AI-generated content.
- **Nodes Involved:**  
  - *Strava*  
- **Node Details:**  
  - **Strava**  
    - Type: Strava node  
    - Role: Performs an update operation on the Strava activity identified by the trigger.  
    - Configuration:  
      - Operation: `update`  
      - Activity ID: Dynamically referenced from Strava Trigger output (`object_id`).  
      - Fields updated: `name` and `description` with AI-generated values.  
    - Input: From "Edit Fields"  
    - Output: None (end node)  
    - Credentials: Same Strava OAuth2 credentials as trigger.  
    - Failures: Permission errors, API limits, invalid activity ID, network failure.  
    - Sticky Note: Describes auto-update of Strava activity.

---

### 3. Summary Table

| Node Name          | Node Type                          | Functional Role                             | Input Node(s)         | Output Node(s)           | Sticky Note                                                                                       |
|--------------------|----------------------------------|---------------------------------------------|-----------------------|--------------------------|-------------------------------------------------------------------------------------------------|
| Strava Trigger     | Strava Trigger                   | Entry point, listens for new activities     | None                  | Code                     | Step 1 – Strava Trigger: Monitors new activities; retrieves raw data.                            |
| Code               | Code (JavaScript)                | Adds/modifies fields in activity data       | Strava Trigger        | Combine Everything       | Step 2a – Data Processing (Code): Adds fields for AI prep.                                      |
| Combine Everything | Code (JavaScript)                | Flattens nested JSON into readable text     | Code                  | Strava Social Manager    | Step 2b – Data Flattening: Prepares data for AI input.                                          |
| Strava Social Manager | Langchain Agent                | Sends data to AI; generates title/desc      | Combine Everything    | Edit Fields              | Step 3 – AI Generation: Sends prompt and data to DeepSeek AI for creative results.               |
| OpenRouter Chat Model | Langchain LM Chat Model        | AI language model provider (DeepSeek AI)    | Strava Social Manager | Strava Social Manager    |                                                                                                 |
| Edit Fields        | Set                              | Extracts title and description from AI text | Strava Social Manager | Strava                  | Step 4 – Extracting AI Results: Parses AI output into fields.                                   |
| Strava             | Strava                          | Updates Strava activity with AI content      | Edit Fields           | None                     | Step 5 – Auto Update: Updates activity title and description on Strava.                         |
| Sticky Note        | Sticky Note                     | Documentation and instructions               | None                  | None                     | Step 0 – OAuth2 setup for Strava API connection.                                               |
| Sticky Note1       | Sticky Note                     | Documentation for Strava Trigger             | None                  | None                     | Step 1 – Explains Strava Trigger role.                                                          |
| Sticky Note2       | Sticky Note                     | Documentation for data processing nodes      | None                  | None                     | Step 2 – Explains Code and Combine Everything nodes.                                            |
| Sticky Note3       | Sticky Note                     | Documentation for AI generation node         | None                  | None                     | Step 3 – Describes AI prompt and expected output.                                              |
| Sticky Note4       | Sticky Note                     | Documentation for extracting AI results      | None                  | None                     | Step 4 – Explains Set node extraction logic.                                                    |
| Sticky Note5       | Sticky Note                     | Documentation for Strava update node         | None                  | None                     | Step 5 – Describes automatic update of Strava activity.                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Strava OAuth2 Credentials:**
   - In n8n, go to Credentials > Strava OAuth2 API.
   - Enter your Client ID and Client Secret from the Strava developer portal.
   - Authorize scopes: `activity:read_all` and `activity:write`.
   - Save credentials as e.g. “Strava account”.

2. **Add Strava Trigger Node:**
   - Node type: Strava Trigger.
   - Configure Event: `create`.
   - Object: `activity`.
   - Enable webhook with a unique webhook ID.
   - Assign Strava OAuth2 credentials created in step 1.
   - This node will automatically receive new activity data.

3. **Add Code Node for Data Enrichment:**
   - Node type: Code.
   - Connect input from “Strava Trigger”.
   - Use JavaScript code to loop through all input items and add a custom field `myNewField` with value 1.
   - Output all modified items.

4. **Add Code Node to Flatten Data ("Combine Everything"):**
   - Node type: Code.
   - Connect input from Code node above.
   - Implement a recursive function in JavaScript that flattens nested JSON objects into a single string with lines formatted as `key: value`.
   - Iterate all input items, concatenate flattened strings with a separator line `---`.
   - Return a single JSON object with the merged string in a field named `data`.

5. **Add Langchain Agent Node ("Strava Social Manager") for AI Generation:**
   - Node type: Langchain Agent.
   - Connect input from "Combine Everything".
   - Set prompt text to instruct AI to generate a catchy title (max 10 words) and a detailed personalized description (3-5 sentences) about the cycling activity.
   - Embed flattened data with variable `{{ $json.data }}` in the prompt.
   - Select agent type: `conversationalAgent`.
   - Assign OpenRouter API credentials configured with DeepSeek AI model (`deepseek-r1:free`).
   - No additional options required.

6. **Add Langchain Chat Model Node (OpenRouter Chat Model):**
   - Node type: Langchain LM Chat Model.
   - Configure model as `deepseek/deepseek-r1:free`.
   - Connect this node as the AI language model provider for the "Strava Social Manager" agent node.

7. **Add Set Node ("Edit Fields") to Extract AI Output:**
   - Node type: Set.
   - Connect input from "Strava Social Manager".
   - Create two string fields:
     - `Name`: Extract first line after `Titre : ` from AI output text using expression:  
       `={{ $json.output.split('\n')[0].split(': ')[1] }}`
     - `Description`: Extract second line after `Description : ` using expression:  
       `={{ $json.output.split('\n')[1].split(': ')[1] }}`
   - This prepares the title and description for updating Strava.

8. **Add Strava Node to Update Activity:**
   - Node type: Strava.
   - Connect input from "Edit Fields".
   - Operation: `update`.
   - Activity ID: Use expression to get ID from trigger node:  
     `={{ $('Strava Trigger').item.json.object_id }}`
   - Update fields:
     - Name: set to `{{$json.Name}}`
     - Description: set to `{{$json.Description}}`
   - Use the same Strava OAuth2 credentials as the trigger node.

9. **Connect Nodes Sequentially:**
   - Strava Trigger → Code (enrichment) → Combine Everything → Strava Social Manager → Edit Fields → Strava update.

10. **Test Workflow:**
    - Trigger by creating a new activity on Strava.
    - Verify that the workflow runs successfully and updates the activity with AI-generated title and description.

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                                                                      |
|----------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Before starting, ensure your Strava OAuth2 credentials include `activity:read_all` and `activity:write` scopes. | Credential setup is crucial for trigger and update nodes to function correctly.                      |
| The AI prompt uses French language instructions but can be adapted to any language or tone desired. | The prompt text is in French to match the cycling community style but can be customized.             |
| DeepSeek AI via OpenRouter provides a free model (`deepseek-r1:free`) specifically for cycling content generation. | OpenRouter model info: https://openrouter.ai/models/deepseek-r1                                    |
| Recursive JSON flattening is implemented to ensure complex nested data is converted into a readable text block for AI context. | Flattening approach inspired by common JSON-to-text conversion methods.                             |
| The workflow relies on real-time webhook triggers from Strava; ensure your n8n instance is publicly accessible to receive them. | Webhook setup and network accessibility are essential for live operation.                           |

---

**Disclaimer:**  
The text and data processed arise exclusively from an automated workflow created with n8n, respecting all applicable content policies. No illegal, offensive, or protected elements are included. All processed data are legal and public.