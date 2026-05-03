Generate AI Descriptions for New Google Sheets Entries with GPT-4.1-mini

https://n8nworkflows.xyz/workflows/generate-ai-descriptions-for-new-google-sheets-entries-with-gpt-4-1-mini-7403


# Generate AI Descriptions for New Google Sheets Entries with GPT-4.1-mini

### 1. Workflow Overview

This workflow automates the generation of AI-written descriptions for new entries added to a Google Sheets spreadsheet. When a new row with a "topic" is added in the designated sheet tab, the workflow triggers, sends the topic to GPT-4.1-mini to generate a structured description, and then updates the original row with this description. Additionally, it logs the update action in a separate "actions" sheet tab to maintain a history of processed entries.

The workflow is logically divided into these blocks:

- **1.1 Input Reception:** Detect new rows added to the Google Sheets "data" tab.
- **1.2 AI Description Generation:** Use GPT-4.1-mini via LangChain nodes to generate a JSON-formatted description for the topic.
- **1.3 Sheet Update and Logging:** Update the original sheet row with the generated description and append an update log in the "actions" tab.
- **1.4 Supporting Documentation:** Sticky notes provide setup instructions, credentials guidance, and contact info.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block detects newly added rows in the "data" tab of the configured Google Sheets document, serving as the workflow's entry trigger.

- **Nodes Involved:**  
  - Row added - Google Sheet

- **Node Details:**  

  - **Row added - Google Sheet**  
    - Type: Google Sheets Trigger  
    - Role: Watches the "data" sheet for any new rows added.  
    - Configuration:  
      - Event: rowAdded  
      - Polling interval: every minute  
      - Sheet: "data" tab (gid=0)  
      - Spreadsheet ID: specific Google Sheets document ID  
      - Credentials: OAuth2 credentials for Google Sheets Trigger  
    - Input: N/A (trigger node)  
    - Output: JSON containing at least the "topic" field of the new row  
    - Edge Cases/Potential Failures:  
      - Authentication errors if credentials expire or are revoked  
      - Polling delays or missed triggers if Google API rate limits or connectivity issues occur  
      - Empty or malformed new rows without a "topic" field  
    - Version: 1

#### 1.2 AI Description Generation

- **Overview:**  
  This block processes the "topic" from the new row, sends it to the OpenAI GPT-4.1-mini model via LangChain nodes to generate a structured description, and parses the AI response.

- **Nodes Involved:**  
  - Description Writer  
  - OpenAI Chat Model  
  - Structured Output Parser

- **Node Details:**  

  - **OpenAI Chat Model**  
    - Type: LangChain OpenAI Chat Model node  
    - Role: Accesses the GPT-4.1-mini model to perform AI text generation.  
    - Configuration:  
      - Model: "gpt-4.1-mini" selected from a list  
      - No additional options set (default parameters)  
      - Credentials: OpenAI API key configured  
    - Input: receives prompt text from "Description Writer" node  
    - Output: raw AI chat completions  
    - Edge Cases:  
      - API key invalid or quota exceeded  
      - Timeout or rate limiting by OpenAI  
      - Unexpected AI output format  
    - Version: 1.2

  - **Description Writer**  
    - Type: LangChain Agent node  
    - Role: Constructs the prompt and sends the "topic" to the AI model for description generation.  
    - Configuration:  
      - Prompt text: dynamically set to the "topic" field from the trigger node (`={{ $json.topic }}`)  
      - System message instructs AI to output a JSON object with a "description" field, e.g.:  
        ```json
        {
          "description": "description"
        }
        ```  
      - Uses a defined prompt type with output parser enabled  
    - Input: receives "topic" from trigger node  
    - Output: AI generated JSON with "description"  
    - Edge Cases:  
      - If the input "topic" is missing or empty, AI might produce irrelevant or empty descriptions  
      - Parser might fail if AI output is malformed or deviates from expected JSON schema  
    - Version: 2.2  
    - Connections: Sends prompt to "OpenAI Chat Model", receives AI response parsed by "Structured Output Parser"

  - **Structured Output Parser**  
    - Type: LangChain Structured Output Parser  
    - Role: Parses the AI response to extract the "description" field as JSON.  
    - Configuration:  
      - JSON schema example provided to validate output structure  
    - Input: AI raw response from "OpenAI Chat Model"  
    - Output: JSON object with "description" key  
    - Edge Cases:  
      - Parsing error if AI response is not valid JSON or schema mismatch  
    - Version: 1.3

#### 1.3 Sheet Update and Logging

- **Overview:**  
  Updates the original Google Sheets row with the AI-generated description and appends an entry to the "actions" sheet to document the update.

- **Nodes Involved:**  
  - Update row in sheet  
  - Append row in sheet

- **Node Details:**  

  - **Update row in sheet**  
    - Type: Google Sheets node (Update operation)  
    - Role: Updates the existing row in the "data" tab with the generated description.  
    - Configuration:  
      - Operation: update  
      - Document ID: same as trigger spreadsheet  
      - Sheet Name: "data" tab (gid=0)  
      - Columns mapped:  
        - topic: copied from trigger node (`={{ $('Row added - Google Sheet').item.json.topic }}`)  
        - description: from AI output (`={{ $json.output.description }}`)  
      - Matching column: "topic" to identify which row to update  
      - Credentials: Google Sheets OAuth2  
    - Input: receives AI description and original topic  
    - Output: confirmation of updated row  
    - Edge Cases:  
      - Matching failure if "topic" is duplicated or missing, causing incorrect row update  
      - API or permission errors updating the sheet  
    - Version: 4.7

  - **Append row in sheet**  
    - Type: Google Sheets node (Append operation)  
    - Role: Appends a log entry to the "actions" tab documenting the update action.  
    - Configuration:  
      - Operation: append  
      - Document ID: same spreadsheet  
      - Sheet Name: "actions" tab (gid=196803262)  
      - Columns mapped:  
        - Update: text stating the topic that was added, e.g., `={{ $('Row added - Google Sheet').item.json.topic }} was added`  
      - Credentials: Google Sheets OAuth2  
    - Input: topic from trigger node  
    - Output: confirmation of appended row  
    - Edge Cases:  
      - API or permission errors appending to the sheet  
      - Sheet structure changes causing mapping issues  
    - Version: 4.7

#### 1.4 Supporting Documentation

- **Overview:**  
  Provides user guidance, setup instructions, and contact information embedded as sticky notes for ease of use and customization.

- **Nodes Involved:**  
  - Sticky Note16  
  - Sticky Note  
  - Sticky Note1

- **Node Details:**  

  - **Sticky Note16**  
    - Type: Sticky Note  
    - Content: Contact info for help or customization requests, including email and LinkedIn link.  
    - Position: top-left area, easy visibility  
    - Edge Cases: None  

  - **Sticky Note**  
    - Type: Sticky Note  
    - Content: Detailed instructions for setting up Google Sheets and enabling Google API access (including links to Google Sheets template and Google Cloud Console).  
    - Position: near the workflow start area for onboarding  
    - Edge Cases: None  

  - **Sticky Note1**  
    - Type: Sticky Note  
    - Content: Instructions on importing workflow JSON, configuring OpenAI and Google credentials, and setting up the trigger node.  
    - Position: near the trigger and AI nodes  
    - Edge Cases: None  

---

### 3. Summary Table

| Node Name               | Node Type                                    | Functional Role                        | Input Node(s)               | Output Node(s)                            | Sticky Note                                                   |
|-------------------------|----------------------------------------------|--------------------------------------|-----------------------------|------------------------------------------|---------------------------------------------------------------|
| Row added - Google Sheet | Google Sheets Trigger                        | Detects new rows in "data" sheet     | N/A                         | Description Writer                       | See Sticky Note1 and Sticky Note for setup instructions       |
| Description Writer       | LangChain Agent                              | Sends topic to GPT-4.1-mini for description generation | Row added - Google Sheet     | Update row in sheet, Append row in sheet | See Sticky Note1 for configuration details                    |
| OpenAI Chat Model        | LangChain OpenAI Chat Model                  | Invokes GPT-4.1-mini model           | Description Writer           | Description Writer (AI response)          | See Sticky Note1 for OpenAI credentials setup                 |
| Structured Output Parser | LangChain Structured Output Parser           | Parses AI output JSON                | OpenAI Chat Model            | Description Writer (parsed output)         |                                                               |
| Update row in sheet      | Google Sheets (Update)                        | Updates original row with AI description | Description Writer           | N/A                                      |                                                               |
| Append row in sheet      | Google Sheets (Append)                        | Logs update action in "actions" tab | Description Writer           | N/A                                      |                                                               |
| Sticky Note16            | Sticky Note                                  | Contact info for help/customization  | N/A                         | N/A                                      | Contact: robert@ynteractive.com, LinkedIn: https://www.linkedin.com/in/robert-breen-29429625/ |
| Sticky Note              | Sticky Note                                  | Setup instructions for Google Sheets | N/A                         | N/A                                      | Includes links to Google Sheets template and Google Cloud Console |
| Sticky Note1             | Sticky Note                                  | Workflow import and credential setup instructions | N/A                         | N/A                                      | Detailed step-by-step import and setup guidance               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Sheets with Required Tabs:**
   - Create a spreadsheet with two tabs:  
     - "data" tab with columns: `topic` (A1), `description` (B1)  
     - "actions" tab with column: `Update` (A1)  
   - Note your spreadsheet ID and GIDs for the tabs.

2. **Enable Google APIs:**
   - In Google Cloud Console:
     - Enable "Google Sheets API" and "Google Drive API".
     - Create OAuth2 credentials for n8n to access Google Sheets.

3. **Set up Google Sheets Trigger Node:**
   - Add a "Google Sheets Trigger" node.  
   - Configure with your OAuth2 credentials.  
   - Select your spreadsheet and the "data" sheet (gid=0).  
   - Set event to "rowAdded".  
   - Set polling interval to every minute.

4. **Add LangChain OpenAI Chat Model Node:**
   - Add an "OpenAI Chat Model" node.  
   - Select the model "gpt-4.1-mini".  
   - Configure with your OpenAI API credentials.

5. **Add LangChain Structured Output Parser Node:**
   - Add a "Structured Output Parser" node.  
   - Set JSON schema example to:  
     ```json
     {
       "description": "description"
     }
     ```

6. **Add LangChain Agent Node (Description Writer):**
   - Add a "LangChain Agent" node.  
   - Set prompt text to: `={{ $json.topic }}` (dynamic from trigger).  
   - Set system message to instruct generating description with output JSON format as above.  
   - Enable output parser and link to "Structured Output Parser" node.

7. **Connect Nodes for AI Processing:**
   - Link "Row added - Google Sheet" output to "Description Writer" input.  
   - Connect "Description Writer" AI prompt to "OpenAI Chat Model".  
   - Connect "OpenAI Chat Model" output to "Structured Output Parser".  
   - Connect "Structured Output Parser" parsed output back to "Description Writer".

8. **Add Google Sheets Update Row Node:**
   - Add a "Google Sheets" node with operation "update".  
   - Configure with the same spreadsheet and "data" sheet.  
   - Map columns:  
     - "topic" from trigger node's `topic` field  
     - "description" from AI output's `description` field  
   - Set matching column to "topic" to identify the row to update.  
   - Use Google Sheets OAuth2 credentials.

9. **Add Google Sheets Append Row Node:**
   - Add a "Google Sheets" node with operation "append".  
   - Configure with the same spreadsheet and "actions" sheet.  
   - Map column "Update" to: `={{ $('Row added - Google Sheet').item.json.topic }} was added`  
   - Use Google Sheets OAuth2 credentials.

10. **Connect AI Output to Sheet Updates:**
    - Connect "Description Writer" output to both "Update row in sheet" and "Append row in sheet" nodes.

11. **Add Sticky Notes for Documentation (Optional):**
    - Add sticky notes with setup instructions, contact info, and links as per your organizational needs.

12. **Test Workflow:**
    - Add a new row with a "topic" in the "data" sheet.  
    - The workflow should trigger, call GPT-4.1-mini, update the "description" in the same row, and append a log entry in the "actions" sheet.

---

### 5. General Notes & Resources

| Note Content                                                                                                                           | Context or Link                                                                                                              |
|----------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------|
| This workflow uses LangChain nodes for better structured prompting and output parsing with OpenAI GPT models.                           | LangChain in n8n documentation and node repository                                                                          |
| Google Sheets template for ease of setup: [AI Description Generator Template](https://docs.google.com/spreadsheets/d/1d8O_Iwg5UtDjHM22zWHMegORatYzaQUYmzrcYnG5OSs/edit?usp=sharing) | Pre-configured spreadsheet with required tabs and columns                                                                    |
| Detailed Google API enablement instructions and OAuth2 setup are critical for seamless integration.                                    | Google Cloud Console: https://console.cloud.google.com/                                                                       |
| Contact for help or customization: robert@ynteractive.com, LinkedIn: https://www.linkedin.com/in/robert-breen-29429625/                | Provided in Sticky Note16                                                                                                     |
| Polling triggers may be subject to Google API rate limits and n8n server limitations; consider webhook alternatives if available.      | n8n documentation on triggers                                                                                                |

---

**Disclaimer:** The provided text derives exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.