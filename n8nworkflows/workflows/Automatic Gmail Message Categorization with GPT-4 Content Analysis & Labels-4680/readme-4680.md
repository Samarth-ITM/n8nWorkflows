Automatic Gmail Message Categorization with GPT-4 Content Analysis & Labels

https://n8nworkflows.xyz/workflows/automatic-gmail-message-categorization-with-gpt-4-content-analysis---labels-4680


# Automatic Gmail Message Categorization with GPT-4 Content Analysis & Labels

### 1. Workflow Overview

This workflow automates the categorization of incoming Gmail messages using AI content analysis powered by OpenAI’s GPT-4. It targets Gmail users and teams overwhelmed by high email volumes, including sales, project management, and support teams who want to streamline inbox organization by automatically labeling emails based on their content intent.

The workflow’s logic is divided into the following blocks:

- **1.1 Gmail Input Reception:** Triggers on new incoming Gmail messages and fetches full email content for analysis.
- **1.2 AI Content Categorization:** Sends the email text to OpenAI GPT-4 to classify the message into predefined label categories.
- **1.3 Label Processing & Mapping:** Parses the AI output, maps AI-assigned label names to existing Gmail label IDs.
- **1.4 Label Aggregation & Application:** Aggregates relevant label IDs and applies these labels to the original Gmail message.

---

### 2. Block-by-Block Analysis

#### 2.1 Gmail Input Reception

**Overview:**  
This block listens for new Gmail messages and retrieves their full content to prepare for AI analysis.

**Nodes Involved:**  
- Gmail trigger  
- Get message content

**Node Details:**

- **Gmail trigger**  
  - Type: Gmail Trigger Node  
  - Role: Watches Gmail inbox and triggers the workflow on new incoming emails.  
  - Configuration: Polls every minute (configurable polling interval). No filters applied, so all messages trigger the workflow.  
  - Input/Output: No input; outputs message metadata including message ID and partial preview text.  
  - Failure Modes: Gmail API rate limits, OAuth token expiration, network timeouts.  
  - Credentials: Requires Gmail OAuth2 credentials.  
  - Notes: Polling interval should be adjusted to balance responsiveness and API usage.

- **Get message content**  
  - Type: Gmail Node (Get operation)  
  - Role: Fetches full email content and metadata using the message ID from the trigger.  
  - Configuration: Uses message ID from the Gmail trigger node (`{{$json.id}}`), retrieves complete message details (`simple=false`).  
  - Input: Message ID from trigger node.  
  - Output: Full message JSON including body text, subject, sender, recipients.  
  - Failure Modes: Invalid message ID, Gmail API errors, OAuth token issues.  
  - Credentials: Same Gmail OAuth2 credentials as trigger node.

---

#### 2.2 AI Content Categorization

**Overview:**  
This block analyzes the email text using GPT-4 to assign one or more intent-based labels.

**Nodes Involved:**  
- Assign labels for message (LangChain LLM Chain)  
- OpenAI GPT-4 Based (LangChain LLM Chat OpenAI)  
- JSON Parser

**Node Details:**

- **Assign labels for message**  
  - Type: LangChain Chain LLM Node  
  - Role: Prepares and sends the prompt to the OpenAI GPT-4 model, requesting classification of the email text into predefined labels.  
  - Configuration:  
    - Text input is the email body text (`{{$node["Gmail trigger"].item.json.text}}`).  
    - Prompt defines four labels: "Quotation", "Project progress", "Inquiry", and "Notification" with their definitions.  
    - Instructs AI to return only JSON with label names (array).  
  - Input: Email text from Gmail trigger (note: uses partial text, not full content from "Get message content" - possibly an oversight or intentional).  
  - Output: AI response with JSON containing assigned labels.  
  - Failure Modes: OpenAI API errors, prompt formatting issues, API rate limits, empty or ambiguous email text.  
  - Notes: Prompt should be customized if label categories change.

- **OpenAI GPT-4 Based**  
  - Type: LangChain LLM Chat OpenAI Node  
  - Role: Executes the actual call to OpenAI GPT-4 Turbo Preview model.  
  - Configuration: GPT-4 Turbo Preview model with temperature set to 0 for deterministic responses; output expected in JSON object format.  
  - Input: Receives prompt from "Assign labels for message" node.  
  - Output: Raw AI response passed downstream for parsing.  
  - Credentials: OpenAI API key required.  
  - Failure Modes: API key issues, rate limits, network errors.

- **JSON Parser**  
  - Type: LangChain Output Parser Structured Node  
  - Role: Parses AI output into structured JSON based on a defined JSON schema validating presence and correctness of labels array.  
  - Configuration:  
    - Schema requires an object with a "labels" array property containing strings restricted to the four predefined labels.  
  - Input: AI raw output JSON string.  
  - Output: Validated structured JSON containing assigned labels.  
  - Failure Modes: Parsing errors if AI response is malformed or does not conform to schema, schema mismatch if labels change without updating schema.

---

#### 2.3 Label Processing & Mapping

**Overview:**  
This block converts the AI-returned label names into corresponding Gmail label IDs by merging with the list of all existing Gmail labels.

**Nodes Involved:**  
- Set label values  
- Split out assigned labels  
- Get all labels  
- Merge corresponding labels

**Node Details:**

- **Set label values**  
  - Type: Set Node  
  - Role: Assigns the parsed labels (from JSON Parser) to a workflow variable for further processing.  
  - Configuration: Creates a new array property `labels` with values from the JSON parser output (`{{$json.output.labels}}`).  
  - Input: Parsed JSON labels.  
  - Output: Sets data for downstream nodes.  
  - Failure Modes: Empty or null input.

- **Split out assigned labels**  
  - Type: Split Out Node  
  - Role: Splits the `labels` array into individual items to process each label separately.  
  - Configuration: Splits on the `labels` field.  
  - Input: Array of label names.  
  - Output: Individual label names as separate items.  
  - Failure Modes: Empty arrays produce no output.

- **Get all labels**  
  - Type: Gmail Node (Label resource)  
  - Role: Retrieves all Gmail labels for the account, including their names and IDs.  
  - Configuration: Returns all labels (`returnAll=true`).  
  - Input: None (triggered by Set label values).  
  - Output: List of all Gmail labels with metadata.  
  - Failure Modes: Gmail API errors, OAuth issues.

- **Merge corresponding labels**  
  - Type: Merge Node  
  - Role: Joins split AI-assigned label names with Gmail label list based on matching label names, effectively mapping label names to their Gmail IDs.  
  - Configuration: Merge mode "combine" with advanced merge by matching the Gmail label `name` field to the AI-assigned `labels` field.  
  - Input: Two streams - split labels and all Gmail labels.  
  - Output: Combined data with label names and their corresponding Gmail IDs.  
  - Failure Modes: No matches found if label names differ between AI output and Gmail labels; inconsistent casing or whitespace can cause mismatches.

---

#### 2.4 Label Aggregation & Application

**Overview:**  
This block aggregates the Gmail label IDs into an array and updates the original Gmail message by applying these labels.

**Nodes Involved:**  
- Aggregate label IDs  
- Add labels to message

**Node Details:**

- **Aggregate label IDs**  
  - Type: Aggregate Node  
  - Role: Collects all matched label IDs into a single array to prepare for batch application.  
  - Configuration: Aggregates the `id` field from merged labels into a new array field called `label_ids`.  
  - Input: Merged labels with Gmail label IDs.  
  - Output: Single item with an array of label IDs.  
  - Failure Modes: Empty input results in no label IDs aggregated.

- **Add labels to message**  
  - Type: Gmail Node (Add Labels operation)  
  - Role: Applies the aggregated Gmail label IDs to the original email message.  
  - Configuration:  
    - Uses `label_ids` from the aggregate node.  
    - Uses the message ID from the Gmail trigger node.  
    - Operation: "addLabels".  
  - Input: Email message ID and label IDs.  
  - Output: Gmail API response confirming label application.  
  - Failure Modes: API errors, invalid label IDs, token expiration.  
  - Credentials: Gmail OAuth2.

---

### 3. Summary Table

| Node Name                | Node Type                          | Functional Role                           | Input Node(s)            | Output Node(s)                      | Sticky Note                                                                                                                                                                |
|--------------------------|----------------------------------|-----------------------------------------|--------------------------|------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Sticky Note6             | Sticky Note                      | Annotation for Aggregation & Update     |                          |                                    | ### Aggregate labels and update to message<br>Create array of label IDs and add to the desired email message in Gmail.                                                    |
| Gmail trigger            | Gmail Trigger                   | Trigger on new Gmail messages            |                          | Get message content                | ### Gmail Trigger<br>Get new incoming message from Gmail. <br>⚠️ Set up the time (polling) interval based on your needs.                                                  |
| Get message content      | Gmail Node (Get operation)       | Retrieve full email content               | Gmail trigger             | Assign labels for message          | ### Get message content<br>Based on Gmail message ID retrieve body content of the email and pass it to AI chain.                                                           |
| Assign labels for message| LangChain Chain LLM              | Send email text to OpenAI for labeling   | Get message content       | Set label values                   | ### Assign labels<br>Let the AI decide which labels suit the best content of the message.<br>⚠️ **Remember to edit system prompt** - modify label names and instructions. |
| OpenAI GPT-4 Based       | LangChain LLM Chat OpenAI        | Execute OpenAI GPT-4 call                 | Assign labels for message | Assign labels for message (ai_languageModel) |                                                                                                                                                                            |
| JSON Parser              | LangChain Output Parser Structured| Parse AI JSON output                      | Assign labels for message | Set label values                   | ### JSON schema<br>Edit JSON schema and label names according to your needs.<br>⚠️ **Label names in system prompt and JSON schema should be the same.**                   |
| Set label values         | Set Node                        | Store parsed labels in workflow variable | JSON Parser               | Split out assigned labels, Get all labels |                                                                                                                                                                            |
| Get all labels           | Gmail Node (Label resource)      | Retrieve all Gmail labels                 | Set label values          | Merge corresponding labels         |                                                                                                                                                                            |
| Split out assigned labels| Split Out Node                  | Split label array into individual labels | Set label values          | Merge corresponding labels         |                                                                                                                                                                            |
| Merge corresponding labels| Merge Node                    | Join AI labels with Gmail labels by name | Get all labels, Split out assigned labels | Aggregate label IDs          | ### Merge labels<br>Combine labels retrieved from Gmail account and assigned by AI together.                                                                                |
| Aggregate label IDs      | Aggregate Node                  | Collect label IDs into one array          | Merge corresponding labels | Add labels to message              |                                                                                                                                                                            |
| Add labels to message    | Gmail Node (Add Labels operation)| Apply labels to the Gmail message         | Aggregate label IDs, Gmail trigger |                                |                                                                                                                                                                            |
| Sticky Note3             | Sticky Note                      | Workflow overview and instructions        |                          |                                    | ## Intelligent Gmail Labeling: Categorize Emails by Content Intent Using OpenAI<br>...long descriptive note about use case and setup.                                      |
| Sticky Note13            | Sticky Note                      | Gmail Trigger explanation                  |                          |                                    | ### Gmail Trigger<br>Get new incoming message from Gmail. <br>⚠️ Set up the time (polling) interval based on your needs.                                                  |
| Sticky Note14            | Sticky Note                      | JSON Parser schema explanation             |                          |                                    | ###<br>JSON schema<br>Edit JSON schema and label names according to your needs.<br>⚠️ **Label names in system prompt and JSON schema should be the same.**               |
| Sticky Note15            | Sticky Note                      | Merge labels explanation                    |                          |                                    | ### Merge labels<br>Combine labels retrieved from Gmail account and assigned by AI together.                                                                                |
| Sticky Note16            | Sticky Note                      | Get message content explanation             |                          |                                    | ### Get message content<br>Based on Gmail message ID retrieve body content of the email and pass it to AI chain.                                                           |
| Sticky Note17            | Sticky Note                      | Assign labels explanation                    |                          |                                    | ### Assign labels<br>Let the AI decide which labels suit the best content of the message.<br>⚠️ **Remember to edit system prompt** - modify label names and instructions. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Gmail Trigger node:**  
   - Type: Gmail Trigger  
   - Configuration: Set polling to "Every Minute" (or as preferred). No filters to catch all incoming emails.  
   - Credentials: Link your Gmail OAuth2 credentials.

2. **Create Get message content node:**  
   - Type: Gmail Node (operation: get message)  
   - Connect input from Gmail Trigger node.  
   - Set Message ID parameter to `={{ $json["id"] }}` (message ID from trigger).  
   - Set `simple` to false to get full message content.  
   - Credentials: Gmail OAuth2.

3. **Create Assign labels for message node:**  
   - Type: LangChain Chain LLM  
   - Connect input from "Get message content".  
   - Parameter "text": Use expression `={{ $node["Gmail trigger"].item.json.text }}` (or adjust to use full message body if desired).  
   - Define prompt with instructions to categorize email into the labels: Quotation, Project progress, Inquiry, Notification.  
   - Set prompt type to "define" with output parser enabled.  
   - Connect to OpenAI GPT-4 node for language model execution.

4. **Create OpenAI GPT-4 Based node:**  
   - Type: LangChain LLM Chat OpenAI  
   - Model: Select "gpt-4-turbo-preview" or appropriate GPT-4 model.  
   - Set temperature to 0 for deterministic outputs.  
   - Response format: JSON object.  
   - Credentials: Connect OpenAI API credentials.  
   - Connect output back to "Assign labels for message" node as language model.

5. **Create JSON Parser node:**  
   - Type: LangChain Output Parser Structured  
   - Input: Connect output from "Assign labels for message".  
   - Define JSON schema: Object with required "labels" property which is an array of strings, allowed values: "Inquiry", "Quotation", "Project progress", "Notification".  
   - Output: Parsed labels JSON.

6. **Create Set label values node:**  
   - Type: Set Node  
   - Input: Connect from JSON Parser.  
   - Create a new field `labels` with value expression `={{ $json.output.labels }}` to store label names array.

7. **Create Split out assigned labels node:**  
   - Type: Split Out Node  
   - Input: Connect from Set label values.  
   - Field to split out: `labels`.

8. **Create Get all labels node:**  
   - Type: Gmail Node (resource: label, operation: get all)  
   - Input: Connect from Set label values (parallel to Split out assigned labels).  
   - Return all labels from Gmail account.  
   - Credentials: Gmail OAuth2.

9. **Create Merge corresponding labels node:**  
   - Type: Merge Node  
   - Input 1: Split out assigned labels (individual label names).  
   - Input 2: Get all labels (Gmail labels list).  
   - Merge mode: Combine  
   - Merge by fields: Match Gmail label `name` with AI label name (field `labels`).  
   - Output: Combined data with label names and corresponding Gmail label IDs.

10. **Create Aggregate label IDs node:**  
    - Type: Aggregate Node  
    - Input: Connect from Merge corresponding labels.  
    - Aggregate field: `id` from Gmail labels.  
    - Output field name: `label_ids` (array of label IDs).

11. **Create Add labels to message node:**  
    - Type: Gmail Node (operation: addLabels)  
    - Input: Connect from Aggregate label IDs.  
    - Parameters:  
      - `labelIds`: `={{ $json.label_ids }}`  
      - `messageId`: `={{ $node["Gmail trigger"].item.json["id"] }}`  
    - Credentials: Gmail OAuth2.

12. **Arrange nodes as per logical flow:**  
    Gmail Trigger → Get message content → Assign labels for message → JSON Parser → Set label values → Split out assigned labels + Get all labels → Merge corresponding labels → Aggregate label IDs → Add labels to message.

13. **Add sticky notes for documentation:**  
    - Add description sticky notes explaining each block’s purpose and instructions as appropriate.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                              | Context or Link                                                                                                     |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------|
| This workflow requires that Gmail labels ("Quotation", "Inquiry", "Project progress", "Notification") are pre-created in the Gmail account for proper mapping and application.                                                            | Setup instructions                                                                                                 |
| The AI prompt and JSON schema must remain in sync regarding label names to avoid parsing errors and mislabeling.                                                                                                                         | Sticky Note14                                                                                                      |
| Polling interval in Gmail trigger should be set carefully to balance API usage and timely processing.                                                                                                                                     | Sticky Note13                                                                                                      |
| Customize the AI prompt to add or adjust label categories to suit your team’s needs.                                                                                                                                                       | Sticky Note17                                                                                                      |
| Uses OpenAI GPT-4 Turbo Preview model for content analysis with temperature 0 for consistent results.                                                                                                                                     | OpenAI GPT-4 Based node configuration                                                                              |
| Workflow credits and detailed explanation provided in the main sticky note describing use cases and instructions.                                                                                                                        | Sticky Note3                                                                                                       |
| For more information on n8n Gmail node and Gmail API limits, refer to official n8n documentation and Google API docs.                                                                                                                   | n8n docs: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.gmail/                                   |

---

**Disclaimer:** The text provided originates exclusively from an automated workflow created using n8n, an integration and automation tool. This processing strictly complies with applicable content policies and contains no illegal, offensive, or protected content. All data handled is legal and publicly accessible.