💬 Daily WhatsApp Group Summarizer – GPT-4o, Google Sheets & Evolution API

https://n8nworkflows.xyz/workflows/---daily-whatsapp-group-summarizer---gpt-4o--google-sheets---evolution-api-3646


# 💬 Daily WhatsApp Group Summarizer – GPT-4o, Google Sheets & Evolution API

### 1. Workflow Overview

This workflow automates the daily summarization of WhatsApp group conversations by integrating Evolution API, Google Sheets, GPT-4o (OpenAI), and Google Drive. It is designed for users who participate in busy WhatsApp groups and want to keep track of conversations efficiently through daily summaries.

**Target Use Cases:**  
- Teams, communities, classes, or any active WhatsApp groups needing daily digest summaries  
- Users wanting to archive conversations and summaries for later reference  
- Automated daily reporting without manual intervention  

**Logical Blocks:**  
- **1.1 Input Reception:** Receives incoming WhatsApp group messages via webhook from Evolution API.  
- **1.2 Data Organization & Validation:** Organizes and validates group data, filters conversations, and checks group validity.  
- **1.3 Data Storage:** Saves collected messages into Google Sheets, organized by date.  
- **1.4 Scheduled Trigger & Group Definition:** Triggers the workflow daily at 11 PM and defines the group for which to generate summaries.  
- **1.5 Data Aggregation & Text Preparation:** Aggregates messages and prepares text for AI summarization.  
- **1.6 AI Summarization:** Uses GPT-4o to generate a structured daily summary of the conversations.  
- **1.7 Summary Storage:** Saves the generated summary as a document in Google Drive.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
Receives WhatsApp group messages pushed from Evolution API via a webhook. This is the entry point for live message data.

- **Nodes Involved:**  
  - Webhook  
  - Organize  
  - Validate Group  

- **Node Details:**  

  - **Webhook**  
    - Type: Webhook (HTTP endpoint)  
    - Role: Receives incoming JSON payloads containing WhatsApp group messages from Evolution API.  
    - Configuration: Default webhook with no specific parameters; expects POST requests.  
    - Inputs: External HTTP requests from Evolution API.  
    - Outputs: Passes data to "Organize" node.  
    - Edge Cases: Invalid payloads, missing fields, or unauthorized requests could cause failures.  
    - Notes: Requires Evolution API to be configured to send messages to this webhook URL.

  - **Organize**  
    - Type: Set  
    - Role: Extracts and organizes relevant fields from the incoming webhook data for further processing.  
    - Configuration: Maps input JSON fields to structured variables (e.g., message content, sender, timestamp).  
    - Inputs: Webhook output.  
    - Outputs: Passes organized data to "Validate Group".  
    - Edge Cases: Missing or malformed data fields may cause errors or incomplete data.

  - **Validate Group**  
    - Type: If  
    - Role: Checks if the incoming message belongs to the target WhatsApp group (by group ID).  
    - Configuration: Conditional check comparing group ID from data with predefined group ID.  
    - Inputs: Organized data from "Organize".  
    - Outputs: If true, proceeds to "Save Conversations"; if false, stops or ignores.  
    - Edge Cases: Incorrect group ID configuration may cause valid messages to be ignored.

---

#### 2.2 Data Storage

- **Overview:**  
Saves validated WhatsApp group messages into a Google Sheets document, organized by date for easy tracking.

- **Nodes Involved:**  
  - Save Conversations (Google Sheets)  

- **Node Details:**  

  - **Save Conversations**  
    - Type: Google Sheets  
    - Role: Appends or updates rows in a Google Sheets spreadsheet with message data.  
    - Configuration: Uses a pre-configured Google Sheets template (linked in setup instructions). Writes messages with metadata (timestamp, sender, message text).  
    - Inputs: Validated message data from "Validate Group".  
    - Outputs: None (end of this branch).  
    - Edge Cases: Google API quota limits, authentication errors, or sheet access issues.  
    - Notes: Requires Google Sheets credentials connected in n8n.

---

#### 2.3 Scheduled Trigger & Group Definition

- **Overview:**  
Triggers the summarization process daily at 11 PM and defines the target WhatsApp group for which to generate the summary.

- **Nodes Involved:**  
  - Starts at 11pm (Schedule Trigger)  
  - Define group (Set)  
  - View conversations + filter (Google Sheets)  
  - Agregar (Aggregate)  

- **Node Details:**  

  - **Starts at 11pm**  
    - Type: Schedule Trigger  
    - Role: Automatically triggers the workflow once daily at 11 PM.  
    - Configuration: Time set to 23:00 daily.  
    - Inputs: None (trigger node).  
    - Outputs: Starts the chain to define group and fetch messages.  
    - Edge Cases: Timezone misconfiguration may cause unexpected trigger times.

  - **Define group**  
    - Type: Set  
    - Role: Sets the group ID and related parameters for filtering messages.  
    - Configuration: Hardcoded or parameterized group ID to specify which WhatsApp group to summarize.  
    - Inputs: Trigger from "Starts at 11pm".  
    - Outputs: Passes group info to "View conversations + filter".  
    - Edge Cases: Incorrect group ID leads to empty or wrong data retrieval.

  - **View conversations + filter**  
    - Type: Google Sheets  
    - Role: Reads stored messages from Google Sheets for the specified group and date.  
    - Configuration: Reads rows filtered by group ID and date (today’s date).  
    - Inputs: Group info from "Define group".  
    - Outputs: Passes filtered messages to "Agregar".  
    - Edge Cases: Empty sheets, API errors, or incorrect filter parameters.

  - **Agregar**  
    - Type: Aggregate  
    - Role: Aggregates the filtered messages into a single text block or structured format for summarization.  
    - Configuration: Aggregates message text fields, possibly concatenating or summarizing raw data.  
    - Inputs: Filtered messages from "View conversations + filter".  
    - Outputs: Passes aggregated text to "Single Text".  
    - Edge Cases: Large message volumes may cause performance issues.

---

#### 2.4 Data Aggregation & Text Preparation

- **Overview:**  
Prepares the aggregated message text for AI summarization by formatting or cleaning the data.

- **Nodes Involved:**  
  - Single Text (Code)  
  - Summary (Chain Summarization)  
  - ChatModel (OpenAI GPT-4o)  
  - Text (Set)  

- **Node Details:**  

  - **Single Text**  
    - Type: Code (JavaScript)  
    - Role: Processes the aggregated messages, possibly cleaning or formatting text for AI input.  
    - Configuration: Custom JavaScript code to prepare input text.  
    - Inputs: Aggregated messages from "Agregar".  
    - Outputs: Passes cleaned text to "Summary".  
    - Edge Cases: Code errors or unexpected input formats.

  - **ChatModel**  
    - Type: Langchain OpenAI Chat Model  
    - Role: Interfaces with GPT-4o to generate a summary based on the prepared text.  
    - Configuration: Uses OpenAI API key, model set to GPT-4o, with prompt templates for summarization.  
    - Inputs: Text from "Single Text" via "Summary" node.  
    - Outputs: AI-generated summary text.  
    - Edge Cases: API rate limits, authentication failures, or prompt errors.

  - **Summary**  
    - Type: Langchain Chain Summarization  
    - Role: Orchestrates the summarization chain, sending text to ChatModel and receiving the summary.  
    - Configuration: Uses Langchain summarization chain with GPT-4o as the language model.  
    - Inputs: Prepared text from "Single Text".  
    - Outputs: Passes summary to "Text" node.  
    - Edge Cases: Chain misconfiguration or AI model errors.

  - **Text**  
    - Type: Set  
    - Role: Sets the final summary text and metadata for saving.  
    - Configuration: Stores summary output and prepares data for Google Drive upload.  
    - Inputs: Summary from "Summary".  
    - Outputs: Passes data to "Google Drive".  
    - Edge Cases: Missing summary data.

---

#### 2.5 Summary Storage

- **Overview:**  
Saves the AI-generated daily summary as a document in Google Drive for easy access and archiving.

- **Nodes Involved:**  
  - Google Drive  

- **Node Details:**  

  - **Google Drive**  
    - Type: Google Drive  
    - Role: Creates or updates a document file in Google Drive containing the daily summary.  
    - Configuration: Uses Google Drive credentials, specifies folder and file naming conventions (e.g., date-based).  
    - Inputs: Summary text and metadata from "Text".  
    - Outputs: None (end of workflow).  
    - Edge Cases: Google Drive API limits, permission errors, or file naming conflicts.

---

### 3. Summary Table

| Node Name               | Node Type                          | Functional Role                          | Input Node(s)           | Output Node(s)           | Sticky Note                                                                                   |
|-------------------------|----------------------------------|----------------------------------------|-------------------------|--------------------------|----------------------------------------------------------------------------------------------|
| Webhook                 | Webhook                          | Receives WhatsApp messages from API    | External HTTP           | Organize                 |                                                                                              |
| Organize                | Set                              | Extracts and organizes message data    | Webhook                 | Validate Group           |                                                                                              |
| Validate Group          | If                               | Checks if message belongs to target group | Organize                | Save Conversations       |                                                                                              |
| Save Conversations      | Google Sheets                    | Saves messages to Google Sheets         | Validate Group           |                          |                                                                                              |
| Starts at 11pm          | Schedule Trigger                 | Triggers daily summarization at 11 PM  |                         | Define group             |                                                                                              |
| Define group            | Set                              | Sets target WhatsApp group ID           | Starts at 11pm          | View conversations + filter |                                                                                              |
| View conversations + filter | Google Sheets                | Reads messages from Google Sheets       | Define group            | Agregar                  |                                                                                              |
| Agregar                 | Aggregate                       | Aggregates messages for summarization  | View conversations + filter | Single Text             |                                                                                              |
| Single Text             | Code                            | Prepares aggregated text for AI         | Agregar                 | Summary                  |                                                                                              |
| ChatModel               | Langchain OpenAI Chat Model     | Calls GPT-4o to generate summary       | Summary (ai_languageModel) | Summary                 |                                                                                              |
| Summary                 | Langchain Chain Summarization   | Orchestrates AI summarization chain    | Single Text             | Text                     |                                                                                              |
| Text                    | Set                             | Sets final summary text for saving     | Summary                 | Google Drive             |                                                                                              |
| Google Drive            | Google Drive                    | Saves summary document in Google Drive | Text                    |                          |                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: Webhook  
   - Purpose: Receive WhatsApp group messages from Evolution API.  
   - Configuration: Default POST webhook, no authentication (or secure as needed).  
   - Connect output to "Organize".

2. **Create Organize Node**  
   - Type: Set  
   - Purpose: Extract relevant fields from webhook JSON (e.g., message text, sender, timestamp, group ID).  
   - Map fields accordingly.  
   - Connect output to "Validate Group".

3. **Create Validate Group Node**  
   - Type: If  
   - Purpose: Check if message group ID matches target group ID.  
   - Condition: `groupId == "your-target-group-id"` (replace with actual ID).  
   - True output connects to "Save Conversations".  
   - False output ends or ignores.

4. **Create Save Conversations Node**  
   - Type: Google Sheets  
   - Purpose: Append message data to Google Sheets.  
   - Configuration: Connect to Google Sheets credential, select spreadsheet and worksheet (use provided template).  
   - Map fields: date, sender, message text, timestamp.  
   - Connect no further outputs.

5. **Create Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Purpose: Trigger workflow daily at 11 PM.  
   - Configuration: Set time to 23:00 daily.  
   - Connect output to "Define group".

6. **Create Define group Node**  
   - Type: Set  
   - Purpose: Define target WhatsApp group ID for summary.  
   - Set variable `groupId` with the target group ID.  
   - Connect output to "View conversations + filter".

7. **Create View conversations + filter Node**  
   - Type: Google Sheets  
   - Purpose: Read messages from Google Sheets filtered by group ID and current date.  
   - Configuration: Use Google Sheets credential, select spreadsheet and worksheet.  
   - Apply filter on group ID and date (today).  
   - Connect output to "Agregar".

8. **Create Agregar Node**  
   - Type: Aggregate  
   - Purpose: Aggregate message texts into a single text block.  
   - Configuration: Aggregate on message text column, concatenate with line breaks or other separator.  
   - Connect output to "Single Text".

9. **Create Single Text Node**  
   - Type: Code (JavaScript)  
   - Purpose: Clean or format aggregated text for AI input.  
   - Sample code: concatenate messages, remove unwanted characters, etc.  
   - Connect output to "Summary".

10. **Create Summary Node**  
    - Type: Langchain Chain Summarization  
    - Purpose: Use GPT-4o to generate a structured summary.  
    - Configuration: Set language model to GPT-4o, configure prompt template for daily summary.  
    - Connect AI language model output to "ChatModel".  
    - Connect main output to "Text".

11. **Create ChatModel Node**  
    - Type: Langchain OpenAI Chat Model  
    - Purpose: Interface with OpenAI GPT-4o API.  
    - Configuration: Provide OpenAI API key credential, select GPT-4o model.  
    - Connect output to "Summary".

12. **Create Text Node**  
    - Type: Set  
    - Purpose: Store final summary text and metadata for saving.  
    - Set variables: summary text, date, filename for Google Drive.  
    - Connect output to "Google Drive".

13. **Create Google Drive Node**  
    - Type: Google Drive  
    - Purpose: Save summary as a document in Google Drive.  
    - Configuration: Connect Google Drive credential, specify folder and file name (e.g., "Summary YYYY-MM-DD.docx").  
    - Map content to document body.  
    - No further outputs.

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                                                                      |
|----------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Google Sheets template for storing messages: [Google Sheets Template](https://docs.google.com/spreadsheets/d/1eLLLHHdkkaBjtu2qtqqwrLBbcGM49kwZq-TMky5y5Oc/edit?usp=sharing) | Required for message storage and filtering.                                                        |
| Workflow creator contact: WhatsApp +5517991557874                                                  | For customization or hiring services.                                                              |
| Workflow sales page: https://iloveflows.gumroad.com                                               | Purchase workflows or templates.                                                                   |
| Compatible with both n8n Cloud and self-hosted instances                                          | Credentials remain secure within n8n.                                                              |
| Evolution API must be configured to send WhatsApp group messages to the webhook URL                | Essential for receiving live data.                                                                 |
| OpenAI GPT-4o model used for AI summarization                                                     | Requires OpenAI API key with access to GPT-4o.                                                     |
| Scheduling node triggers daily summary generation at 11 PM                                        | Timezone should be verified to match user’s local time.                                            |

---

This documentation provides a complete understanding of the workflow’s structure, logic, and configuration to enable reproduction, modification, and troubleshooting.