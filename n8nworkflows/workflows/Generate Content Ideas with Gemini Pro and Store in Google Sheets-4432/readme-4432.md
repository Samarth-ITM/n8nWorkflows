Generate Content Ideas with Gemini Pro and Store in Google Sheets

https://n8nworkflows.xyz/workflows/generate-content-ideas-with-gemini-pro-and-store-in-google-sheets-4432


# Generate Content Ideas with Gemini Pro and Store in Google Sheets

### 1. Workflow Overview

This workflow is designed to help marketing professionals and content creators generate content ideas automatically using Google's Gemini Pro AI model, then store those ideas conveniently in a Google Sheet for further planning and tracking. It is a beginner-friendly, streamlined automation that integrates manual input, AI-powered content generation, processing of AI output, and data storage.

The workflow is logically divided into four main blocks:

- **1.1 Input Reception**: Captures the topic or keyword manually from the user.
- **1.2 AI Processing**: Sends the topic to Google AI (Gemini Pro) to generate content ideas.
- **1.3 Output Formatting**: Parses and formats the raw AI-generated text into individual content ideas.
- **1.4 Data Storage**: Appends each formatted idea as a new row into a designated Google Sheet.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block captures the user's desired topic or keyword for content idea generation. It serves as the workflow's entry point, requiring manual user input in JSON format.

- **Nodes Involved:**  
  - Manual Input

- **Node Details:**

  - **Manual Input**  
    - Type: `manualInput` (Trigger/Start node)  
    - Role: Accepts manual JSON input with a required field `topic` (e.g., `{"topic": "your desired topic"}`).  
    - Configuration: Description instructs user to input the topic keyword.  
    - Inputs: None (starting point)  
    - Outputs: Connects to `Generate Ideas (Google AI)` node  
    - Edge cases/failures: Missing or malformed JSON input; absence of `topic` field would cause downstream nodes to fail or generate irrelevant output. Input validation is manual and should be enforced by users.  
    - Version: Compatible with n8n v1 and above.

#### 2.2 AI Processing

- **Overview:**  
  Uses the Google AI node configured with the Gemini Pro model to generate a list of content ideas based on the provided topic.

- **Nodes Involved:**  
  - Generate Ideas (Google AI)

- **Node Details:**

  - **Generate Ideas (Google AI)**  
    - Type: `google-ai`  
    - Role: Sends prompt to Google Gemini Pro to generate 5 content ideas related to the input topic.  
    - Configuration:  
      - Model: Gemini Pro  
      - Operation: `generateContent`  
      - Prompt: `"Generate 5 content ideas related to the topic: {{ $json.topic }}. Please format each idea as a short sentence or title."`  
      - Requires Google AI API credentials (configured separately)  
    - Inputs: Connected from `Manual Input` node, expects JSON with `topic` field  
    - Outputs: Raw AI response, containing text with multiple newline-separated content ideas  
    - Edge cases/failures:  
      - Authentication failure if Google AI credentials are invalid or missing.  
      - API rate limits or response timeouts.  
      - Unexpected AI output format or empty responses.  
    - Version: Requires n8n nodes version supporting Google AI integration.

#### 2.3 Output Formatting

- **Overview:**  
  Processes the raw multiline string output from the AI, splitting it into individual content ideas to facilitate row-wise insertion into Google Sheets.

- **Nodes Involved:**  
  - Format Ideas

- **Node Details:**

  - **Format Ideas**  
    - Type: `function` (JavaScript code execution)  
    - Role: Splits the multiline string returned by Google AI into an array of individual ideas; trims whitespace and filters out empty lines.  
    - Configuration: Uses a JavaScript code snippet:  
      ```javascript
      return items.map(item => {
        const ideas = item.json.candidates[0].content.parts[0].text.split('\n').filter(idea => idea.trim() !== '');
        return ideas.map(idea => ({ json: { idea: idea.trim() } }));
      }).flat();
      ```  
    - Inputs: Receives raw AI-generated content from `Generate Ideas (Google AI)`  
    - Outputs: Array of items each containing a single idea under `json.idea`  
    - Edge cases/failures:  
      - If AI output format changes, path `candidates[0].content.parts[0].text` may not exist, causing runtime errors.  
      - Empty or malformed text could result in zero output items.  
    - Version: Compatible with any n8n version supporting function nodes.

#### 2.4 Data Storage

- **Overview:**  
  Appends each formatted content idea as a new row in the specified Google Sheet, enabling easy access and management of generated ideas.

- **Nodes Involved:**  
  - Write to Google Sheet

- **Node Details:**

  - **Write to Google Sheet**  
    - Type: `google-sheets`  
    - Role: Appends each idea to column A of the specified Google Sheet.  
    - Configuration:  
      - Operation: `append`  
      - Spreadsheet ID: Must be replaced with user’s actual Google Sheet ID (`YOUR_GOOGLE_SHEET_ID`)  
      - Range: `Sheet1!A:A` (entire first column)  
      - Values: Expression `={{ $json.idea }}` to write each idea string into a separate row  
      - Requires Google Sheets OAuth2 credentials  
    - Inputs: Connected from `Format Ideas` node  
    - Outputs: None (end of workflow)  
    - Edge cases/failures:  
      - Authentication errors if credentials are invalid or expired.  
      - Incorrect spreadsheet ID or insufficient permissions cause failures.  
      - API rate limits or quota errors.  
    - Version: Requires n8n nodes version supporting Google Sheets OAuth2 integration.

---

### 3. Summary Table

| Node Name               | Node Type          | Functional Role                        | Input Node(s)    | Output Node(s)   | Sticky Note                                                                                                             |
|-------------------------|--------------------|-------------------------------------|------------------|------------------|-------------------------------------------------------------------------------------------------------------------------|
| Manual Input            | manualInput        | Captures manual topic input          | None             | Generate Ideas (Google AI) |                                                                                                                         |
| Generate Ideas (Google AI) | google-ai          | Generates content ideas via Gemini Pro | Manual Input     | Format Ideas     | Requires Google AI credentials to be set up in n8n.                                                                     |
| Format Ideas            | function           | Splits AI output into individual ideas | Generate Ideas (Google AI) | Write to Google Sheet | Processes raw text output; splits newline-separated ideas for Google Sheets insertion.                                   |
| Write to Google Sheet   | google-sheets      | Appends ideas as new rows in Google Sheet | Format Ideas     | None             | Remember to replace 'YOUR_GOOGLE_SHEET_ID' with your actual sheet ID; connect your Google Sheets OAuth2 credentials.     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Input node**  
   - Name: `Manual Input`  
   - Type: `manualInput`  
   - Parameters: Add a description instructing the user to input JSON with field `"topic"`. For example, enter `{ "topic": "your desired topic" }`.  
   - No credentials needed.  
   - Position it at the start of your workflow.

2. **Create Google AI node**  
   - Name: `Generate Ideas (Google AI)`  
   - Type: `google-ai`  
   - Parameters:  
     - Model: Select `gemini-pro`  
     - Operation: `generateContent`  
     - Prompt: Enter:  
       ```
       Generate 5 content ideas related to the topic: {{ $json.topic }}. Please format each idea as a short sentence or title.
       ```  
     - Additional Parameters: leave empty  
   - Credentials: Set up and select your Google AI API credentials in n8n.  
   - Connect output of `Manual Input` to input of this node.

3. **Create Function node**  
   - Name: `Format Ideas`  
   - Type: `function`  
   - Parameters: Paste the following JavaScript code:  
     ```javascript
     return items.map(item => {
       const ideas = item.json.candidates[0].content.parts[0].text.split('\n').filter(idea => idea.trim() !== '');
       return ideas.map(idea => ({ json: { idea: idea.trim() } }));
     }).flat();
     ```  
   - Connect output of `Generate Ideas (Google AI)` to input of this node.

4. **Create Google Sheets node**  
   - Name: `Write to Google Sheet`  
   - Type: `google-sheets`  
   - Parameters:  
     - Operation: `append`  
     - Spreadsheet ID: Replace `"YOUR_GOOGLE_SHEET_ID"` with your actual Google Sheet ID (found in the Google Sheets URL).  
     - Range: Set to `Sheet1!A:A` to append to column A.  
     - Values: Use expression: `={{ $json.idea }}` to insert each idea as a new row.  
   - Credentials: Configure and select your Google Sheets OAuth2 credentials.  
   - Connect output of `Format Ideas` to input of this node.

5. **Save and Activate Workflow**  
   - Verify all nodes are connected as:  
     Manual Input → Generate Ideas (Google AI) → Format Ideas → Write to Google Sheet  
   - Test manually by triggering the Manual Input node with a sample topic.

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                                                                      |
|---------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| This workflow is compatible only with self-hosted versions of n8n due to credential and API access requirements. | Workflow description metadata.                                                                     |
| For Google AI credentials setup, refer to Google's official API documentation and n8n credential setup guides. | https://cloud.google.com/ai-platform/docs/setting-up-credentials                                   |
| Replace placeholder `YOUR_GOOGLE_SHEET_ID` with your actual Google Sheet ID to enable proper appending.       | Google Sheets URL example: `https://docs.google.com/spreadsheets/d/YOUR_GOOGLE_SHEET_ID/edit`      |
| Ensure Google Sheets OAuth2 credentials have sufficient permissions to edit the target spreadsheet.            | n8n docs: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.google-sheets/          |

---

**Disclaimer:** The provided text is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.