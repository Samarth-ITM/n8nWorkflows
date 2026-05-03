Capture and Process Ideas with GPT-4o-mini, Notion and Slack Notifications

https://n8nworkflows.xyz/workflows/capture-and-process-ideas-with-gpt-4o-mini--notion-and-slack-notifications-8846


# Capture and Process Ideas with GPT-4o-mini, Notion and Slack Notifications

### 1. Workflow Overview

This workflow, titled **"Capture and Process Ideas with GPT-4o-mini, Notion and Slack Notifications"**, is designed to automate the collection, processing, and notification of product ideas submitted by users. It is targeted at teams or organizations that want to streamline idea intake from various sources and integrate them into a centralized Notion database, while also providing immediate Slack notifications for team transparency.

The workflow is logically divided into four main blocks:

- **1.1 Input Reception:** Captures incoming idea submissions via an HTTP webhook.
- **1.2 AI Processing:** Sends the raw input text to an AI agent (powered by GPT-4o-mini) to parse and structure the idea details.
- **1.3 Data Extraction & Storage:** Extracts structured fields from the AI output using a code node, then adds the data to a Notion database.
- **1.4 Notification:** Sends a confirmation message to a Slack channel to notify the team of the new idea submission.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block listens for incoming HTTP POST requests carrying idea submissions in JSON format. It acts as the entry point of the workflow.

- **Nodes Involved:**  
  - 🌐 Webhook

- **Node Details:**  

  **🌐 Webhook**  
  - Type: Webhook (n8n core node)  
  - Role: Entry trigger node accepting POST requests at a defined path (`webhook-path-here`).  
  - Configuration:  
    - HTTP Method: POST  
    - Path: `webhook-path-here` (replace with actual endpoint)  
    - No authentication or additional options configured.  
  - Inputs: External HTTP POST requests with JSON payload containing at least `text` and `user_id` fields in the body.  
  - Outputs: Forwards the body to the next node as `$json.body`.  
  - Edge Cases / Potential Failures:  
    - Missing or malformed JSON in request body.  
    - Incorrect HTTP method usage.  
    - Unauthorized or spam submissions (no auth configured).  
  - Version: 2.1

---

#### 1.2 AI Processing

- **Overview:**  
  This block sends the incoming text and user ID to an AI agent that processes the input and returns structured data fields: Title, Tags, Submitted By, and Created date in IST timezone.

- **Nodes Involved:**  
  - 🤖 AI Agent  
  - 💬 OpenAI Chat Model (linked as AI language model to AI Agent)

- **Node Details:**  

  **🤖 AI Agent**  
  - Type: LangChain AI Agent node (n8n integration)  
  - Role: To process raw input text and user ID, generating structured output.  
  - Configuration:  
    - Text input constructed dynamically using expression:  
      ```  
      {{ $json.body.text }}  
      User ID: {{ $json.body.user_id }}  
      
      this is the input based on the input you have to give  
      Title, tags, submitted by, created(Actual date in IST) {{ new Date().toISOString() }}  
      ```  
    - Prompt type: "define" (custom prompt instructing the AI to extract specific fields).  
    - Options: default.  
  - Inputs: Receives webhook body JSON.  
  - Outputs: AI-generated text containing details in a structured text format (e.g., lines starting with "Title:", "Tags:", etc.).  
  - Linked Node: Receives language model input from **💬 OpenAI Chat Model** node.  
  - Edge Cases:  
    - AI may return incomplete or malformed output.  
    - Network or API failures with OpenAI.  
    - Date/time formatting inconsistencies.  
  - Version: 2.2

  **💬 OpenAI Chat Model**  
  - Type: LangChain OpenAI Chat Model node  
  - Role: Provides GPT-4o-mini model for the AI Agent to use.  
  - Configuration:  
    - Model: gpt-4o-mini  
    - Options: default  
  - Inputs: Connected as AI language model for **🤖 AI Agent**.  
  - Outputs: Processed AI completions to AI Agent.  
  - Edge Cases:  
    - API rate limits or authentication errors with OpenAI.  
    - Model latency or unexpected responses.  
  - Version: 1.2

---

#### 1.3 Data Extraction & Storage

- **Overview:**  
  This block parses the AI-generated text output into structured JSON fields and adds these as a new page in a Notion database.

- **Nodes Involved:**  
  - 🧑‍💻 Code  
  - 📝 Add to Notion

- **Node Details:**  

  **🧑‍💻 Code**  
  - Type: Code (JavaScript) node  
  - Role: Parses AI output text to extract specific fields: Title, Tags, Submitted By, Created.  
  - Configuration:  
    - JavaScript code uses regex matching to find lines starting with each field name.  
    - Tags are split into an array by commas and trimmed.  
    - Created date removes "IST" suffix for Notion compatibility.  
  - Key expressions:  
    - Regex matches for `Title:\s*(.+)`, `Tags:\s*(.+)`, `Submitted by:\s*(.+)`, `Created:\s*(.+)`  
    - Returns array of cleaned JSON objects with the four fields.  
  - Inputs: Receives AI Agent output in `$json.output`.  
  - Outputs: Structured JSON with fields for next node.  
  - Edge Cases:  
    - Missing fields or malformed AI output may result in nulls or empty arrays.  
    - Date string may be in unexpected format.  
  - Version: 2

  **📝 Add to Notion**  
  - Type: Notion node  
  - Role: Adds a new page to a Notion database with the parsed fields.  
  - Configuration:  
    - Resource: databasePage  
    - Database ID: Must be replaced with actual Notion database ID (placeholder `"YOUR_NOTION_DATABASE_ID"`).  
    - Properties mapped as follows:  
      - Title → Notion title property  
      - Submitted By → rich_text property  
      - Created → date property (timezone Asia/Kolkata, time excluded)  
      - Tags → rich_text property as comma-separated string  
  - Inputs: Receives structured JSON from Code node.  
  - Outputs: Confirmation of item creation for next node.  
  - Edge Cases:  
    - Invalid Notion credentials or permissions.  
    - Incorrect database ID or property keys.  
    - Date formatting errors.  
  - Version: 2

---

#### 1.4 Notification

- **Overview:**  
  Sends a confirmation to a Slack channel that the idea has been successfully added, echoing the idea’s title.

- **Nodes Involved:**  
  - ✅ Send Confirmation (Slack)

- **Node Details:**  

  **✅ Send Confirmation (Slack)**  
  - Type: Slack node (n8n core)  
  - Role: Posts a message to a Slack channel notifying about the new idea submission.  
  - Configuration:  
    - Text message template:  
      ```  
      ✅ Your idea has been added to our Product Ideas database!  
      
      💡 *Idea:* {{ $json.Title }}  
      ```  
    - Channel: #general (configured by channel name)  
    - Authentication: OAuth2 (Slack app credentials must be configured)  
  - Inputs: Receives JSON with Title from Notion node output.  
  - Outputs: Slack API response.  
  - Edge Cases:  
    - Slack authentication failure or token expiration.  
    - Channel not found or insufficient permissions.  
    - Message formatting errors.  
  - Version: 2

---

### 3. Summary Table

| Node Name                 | Node Type                      | Functional Role                            | Input Node(s)        | Output Node(s)           | Sticky Note                                                                                                                                                      |
|---------------------------|--------------------------------|--------------------------------------------|----------------------|--------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------|
| 🌐 Webhook                | Webhook (n8n core)             | Receives HTTP POST idea submissions        | —                    | 🤖 AI Agent              | *Input text comes in via Webhook ($json.body.text + user_id). AI Agent is instructed to output Title, Tags, Submitted By, and Created date in IST.*            |
| 🤖 AI Agent               | LangChain AI Agent node        | Processes input text to structured fields  | 🌐 Webhook            | 🧑‍💻 Code                | *AI Agent processes raw input to extract Title, Tags, Submitted By, Created date (IST).*                                                                        |
| 💬 OpenAI Chat Model      | LangChain OpenAI Chat Model    | Provides GPT-4o-mini model for AI Agent    | — (AI model for AI Agent) | 🤖 AI Agent (ai_languageModel) |                                                                                                                                                                 |
| 🧑‍💻 Code                | Code (JavaScript)              | Parses AI output into JSON fields           | 🤖 AI Agent           | 📝 Add to Notion         | *Code node ensures clean extraction: Title, Tags (array), Submitted By, Created (date without "IST").*                                                          |
| 📝 Add to Notion          | Notion node                   | Adds parsed data as a new page in Notion DB | 🧑‍💻 Code             | ✅ Send Confirmation (Slack) | *Data is added to Notion with mapped properties: Title, Submitted By, Created (clean date), Tags (comma-separated).*                                            |
| ✅ Send Confirmation (Slack) | Slack node                  | Sends Slack notification confirming idea addition | 📝 Add to Notion       | —                        | *Notifies Slack channel with confirmation and echoes idea title.*                                                                                              |
| Sticky Note               | Sticky Note                   | Documentation: Data Handling & Transformation | —                    | —                        | *Input text comes in via Webhook ($json.body.text + user_id). AI Agent is instructed to output Title, Tags, Submitted By, and Created date in IST.*            |
| Sticky Note1              | Sticky Note                   | Documentation: Notion Database Integration  | —                    | —                        | *Data is added into Notion (Ideas DB) with mapped properties: Title, Submitted By, Created (IST without suffix), Tags.*                                        |
| Sticky Note2              | Sticky Note                   | Documentation: Slack Notifications           | —                    | —                        | *Once saved in Notion, the workflow notifies Slack with confirmation and echoes back the submitted idea title.*                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**
   - Add a **Webhook** node.
   - Set HTTP Method to **POST**.
   - Set Path to `"webhook-path-here"` or your desired endpoint.
   - No authentication configured.
   - This node will receive JSON payloads containing at least `text` and `user_id`.

2. **Create OpenAI Chat Model Node**
   - Add **LangChain OpenAI Chat Model** node.
   - Select model **gpt-4o-mini**.
   - Leave options as default.
   - This node will serve as the AI backend for the AI Agent.

3. **Create AI Agent Node**
   - Add **LangChain AI Agent** node.
   - Set **Prompt Type** to **define**.
   - In the **Text** parameter, enter the following expression:  
     ```
     {{ $json.body.text }}
     User ID: {{ $json.body.user_id }}

     this is the input based on the input you have to give 
     Title, tags, submitted by, created(Actual date in IST) {{ new Date().toISOString() }}
     ```
   - Connect the AI Agent's **AI Language Model** input to the **OpenAI Chat Model** node.
   - Connect the Webhook node output to this AI Agent node input.

4. **Create Code Node**
   - Add a **Code** node (JavaScript).
   - Use the following JavaScript code to extract fields from AI output:
     ```js
     const items = $input.all();

     let results = [];

     for (const item of items) {
       const text = item.json.output;

       const titleMatch = text.match(/Title:\s*(.+)/);
       const tagsMatch = text.match(/Tags:\s*(.+)/);
       const submittedByMatch = text.match(/Submitted by:\s*(.+)/);
       const createdMatch = text.match(/Created:\s*(.+)/);

       results.push({
         json: {
           Title: titleMatch ? titleMatch[1].trim() : null,
           Tags: tagsMatch ? tagsMatch[1].split(",").map(tag => tag.trim()) : [],
           "Submitted By": submittedByMatch ? submittedByMatch[1].trim() : null,
           Created: createdMatch ? createdMatch[1].trim() : null
         }
       });
     }

     return results;
     ```
   - Connect the AI Agent node output to this Code node.

5. **Create Notion Node**
   - Add a **Notion** node.
   - Set **Resource** to **databasePage**.
   - Provide your **Notion Database ID** where ideas should be stored.
   - Map the properties as follows:  
     - Title property: Set to expression `{{$json.Title}}`  
     - Submitted By (rich_text): `{{$json["Submitted By"]}}`  
     - Created (date): `{{$json.Created.replace(' IST', '')}}`  
       - Set timezone to **Asia/Kolkata**  
       - Disable **Include Time** (date only)  
     - Tags (rich_text): `{{$json.Tags.join(', ')}}`  
   - Connect the Code node output to this Notion node.

6. **Create Slack Node**
   - Add a **Slack** node.
   - Set **Authentication** to **OAuth2** and configure Slack credentials with relevant scopes to post messages.  
   - Set **Channel** to `#general` (or your preferred channel).  
   - Set message text to:  
     ```
     ✅ Your idea has been added to our Product Ideas database!

     💡 *Idea:* {{ $json.Title }}
     ```
   - Connect the Notion node output to this Slack node.

7. **Connect Nodes**
   - Connect the nodes in this order:  
     `Webhook` → `AI Agent` → `Code` → `Notion` → `Slack`

8. **Test Workflow**
   - Activate the workflow.
   - Send a POST request with JSON body including `text` and `user_id` to the webhook URL.
   - Verify Notion database entry and Slack notification.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                            | Context or Link                                                                                  |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| The workflow is designed to handle idea submissions with timestamps in IST timezone and ensures clean data extraction for Notion integration.                        | Sticky note on Data Handling & Transformation                                                  |
| Notion properties must exactly match the configured property keys in your database schema for successful data insertion (Title, Submitted By, Created, Tags).         | Sticky note on Notion Database Integration                                                     |
| Slack OAuth2 credentials require permissions to post messages in target channels; ensure your Slack app and tokens are correctly set up before running the workflow.   | Sticky note on Slack Notifications                                                             |

---

**Disclaimer:** The provided text and workflow are entirely generated and structured from an automated n8n workflow export. It complies with all content policies and contains no illegal or protected information. All data and integrations are assumed to be public or authorized for use.