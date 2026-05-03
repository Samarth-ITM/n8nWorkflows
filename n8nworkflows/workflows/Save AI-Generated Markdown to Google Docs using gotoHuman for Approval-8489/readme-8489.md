Save AI-Generated Markdown to Google Docs using gotoHuman for Approval

https://n8nworkflows.xyz/workflows/save-ai-generated-markdown-to-google-docs-using-gotohuman-for-approval-8489


# Save AI-Generated Markdown to Google Docs using gotoHuman for Approval

### 1. Workflow Overview

This workflow automates the creation, human review, and saving of an AI-generated content strategy document in Google Docs. It is designed primarily for marketing or content teams who want to draft strategic content with AI assistance, ensure quality and correctness through human approval, and then update an official document stored on Google Drive.

The workflow is logically divided into these main blocks:

- **1.1 Input Reception:** Manual trigger to start the workflow.
- **1.2 AI Content Generation:** Uses contextual documents and an AI agent to generate a content strategy in markdown format.
- **1.3 Human Review:** Sends the generated markdown to a human review platform (gotoHuman), enabling edits and approval.
- **1.4 Post-Approval Processing:** Checks approval status, converts markdown to a file, sets correct MIME type, and updates Google Docs file with the final content.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:** This block initiates the workflow upon manual execution.
- **Nodes Involved:**  
  - When clicking ‘Execute workflow’

- **Node Details:**

  - **When clicking ‘Execute workflow’**  
    - Type: Manual Trigger  
    - Role: Starts the workflow manually when triggered by the user.  
    - Configuration: No parameters; simple manual trigger.  
    - Input: None  
    - Output: Triggers the AI Agent node.  
    - Edge Cases: None significant; user must trigger manually.  
    - Version: n8n-nodes-base.manualTrigger v1

---

#### 1.2 AI Content Generation

- **Overview:** This block retrieves company context documents and uses an AI agent to generate a content strategy in markdown format.
- **Nodes Involved:**  
  - AI Agent  
  - Get company description  
  - Get company strategy  
  - OpenAI Chat Model  
  - Structured Output Parser

- **Node Details:**

  - **Get company description**  
    - Type: Google Docs Tool  
    - Role: Retrieves the company description document for context.  
    - Configuration: Operation set to "get" to fetch document content.  
    - Input: Triggered by AI Agent node as an AI tool input.  
    - Output: Supplies document content data to AI Agent.  
    - Edge Cases: Google Docs API auth errors, missing or inaccessible document.  
    - Version: 2  

  - **Get company strategy**  
    - Type: Google Docs Tool  
    - Role: Retrieves existing company strategy document as additional context.  
    - Configuration: Operation "get" for document content retrieval.  
    - Input: AI Agent node.  
    - Output: Supplies document content for AI processing.  
    - Edge Cases: Same as above.  
    - Version: 2  

  - **OpenAI Chat Model**  
    - Type: LangChain OpenAI Chat Model  
    - Role: Provides AI language model backend (GPT-4.1) for the AI Agent.  
    - Configuration: Model set to "gpt-4.1" (chat-based), no extra options configured.  
    - Input: AI Agent language model input.  
    - Output: AI-generated text.  
    - Edge Cases: OpenAI API rate limits, auth failures, model unavailability.  
    - Version: 1.2  

  - **Structured Output Parser**  
    - Type: LangChain Output Parser (Structured)  
    - Role: Parses AI output to a JSON structure with a field `strategyMarkdown` containing markdown text.  
    - Configuration: Uses a JSON schema example with “strategyMarkdown” string field.  
    - Input: AI Agent output.  
    - Output: Parsed structured JSON for downstream nodes.  
    - Edge Cases: Parsing errors if AI output does not conform to expected JSON schema.  
    - Version: 1.3  

  - **AI Agent**  
    - Type: LangChain AI Agent  
    - Role: Orchestrates the input documents and AI model to generate the markdown content strategy.  
    - Configuration:  
      - Prompt: "Write a content strategy based on the company description and the given overall strategy. Consider the channels LinkedIn, Instagram and Email Newsletter"  
      - Uses the above Google Docs documents as AI tools inputs.  
      - Uses OpenAI Chat Model as language model.  
      - Uses Structured Output Parser to parse output.  
    - Input: Manual trigger initiates this node.  
    - Output: Outputs structured JSON with markdown content.  
    - Edge Cases: Failure if input docs unavailable or AI model fails; expression failures.  
    - Version: 2.2  

---

#### 1.3 Human Review

- **Overview:** Sends the AI-generated markdown content to a human review platform (gotoHuman) where it can be edited and approved. The workflow pauses until a response is received.
- **Nodes Involved:**  
  - Send review request and wait for response  
  - Approved?

- **Node Details:**

  - **Send review request and wait for response**  
    - Type: gotoHuman Node  
    - Role: Sends markdown content to gotoHuman for review and awaits human interaction.  
    - Configuration:  
      - Sends the field `strategyMarkdown` with the generated markdown content.  
      - Uses a predefined review template ID selected from a list (pre-built "Strategy agent" template recommended).  
      - Mapping Mode: Defines fields explicitly below.  
      - Waits via webhook for human response.  
    - Input: AI Agent output (structured JSON).  
    - Output: JSON containing human response and approval status.  
    - Edge Cases: Webhook timeout, network issues, invalid template ID, user abandonment.  
    - Version: 1  

  - **Approved?**  
    - Type: If Node  
    - Role: Checks if the human response status equals "approved".  
    - Configuration:  
      - Condition: `$json.response === "approved"` (case-sensitive, strict validation).  
    - Input: gotoHuman node response.  
    - Output: Branches workflow if approved (true) or not.  
    - Edge Cases: Unrecognized response values, missing response property.  
    - Version: 2.2  

---

#### 1.4 Post-Approval Processing

- **Overview:** After approval, converts the markdown to a file, sets MIME type correctly, and updates the Google Docs file with new content.
- **Nodes Involved:**  
  - Convert to File  
  - Set file type as markdown  
  - Update file

- **Node Details:**

  - **Convert to File**  
    - Type: Convert to File  
    - Role: Converts the markdown string from the gotoHuman review response into a text file format.  
    - Configuration:  
      - Operation: "toText"  
      - Source Property: `responseValues.strategyMarkdown.value` — extracts markdown content from human-reviewed data.  
    - Input: Approved? node (on approval path).  
    - Output: Binary file data containing markdown text.  
    - Edge Cases: Missing or malformed markdown content, conversion failure.  
    - Version: 1.1  

  - **Set file type as markdown**  
    - Type: Code (JavaScript)  
    - Role: Ensures the binary file's MIME type is set to 'text/markdown' for proper handling in Google Drive.  
    - Configuration:  
      - Runs once per item.  
      - Checks if binary data exists, sets `mimeType` to 'text/markdown'.  
    - Input: Convert to File node.  
    - Output: Same item with updated MIME type.  
    - Edge Cases: Missing binary data, code exceptions.  
    - Version: 2  

  - **Update file**  
    - Type: Google Drive  
    - Role: Updates the specified Google Docs file with the new markdown content file.  
    - Configuration:  
      - Operation: "update"  
      - `fileId`: Specified via URL mode (must be configured)  
      - `changeFileContent`: true (content replaced with new markdown file)  
    - Input: Set file type as markdown node.  
    - Output: Confirmation of file update.  
    - Edge Cases: Invalid file ID, permission issues, network errors.  
    - Version: 3  

---

### 3. Summary Table

| Node Name                          | Node Type                           | Functional Role                          | Input Node(s)                        | Output Node(s)                            | Sticky Note                                                                                     |
|-----------------------------------|-----------------------------------|----------------------------------------|------------------------------------|------------------------------------------|------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’  | Manual Trigger                    | Starts workflow manually                | None                               | AI Agent                                 |                                                                                                |
| AI Agent                         | LangChain AI Agent                | Generates AI markdown content strategy | When clicking ‘Execute workflow’   | Send review request and wait for response |                                                                                                |
| Get company description           | Google Docs Tool                  | Retrieves company description document | AI Agent (as AI tool)               | AI Agent                                 |                                                                                                |
| Get company strategy              | Google Docs Tool                  | Retrieves existing company strategy doc | AI Agent (as AI tool)               | AI Agent                                 |                                                                                                |
| OpenAI Chat Model                | LangChain OpenAI Chat Model       | Provides GPT-4.1 language model         | AI Agent (as language model)        | AI Agent                                 |                                                                                                |
| Structured Output Parser          | LangChain Output Parser Structured | Parses AI output to structured JSON     | AI Agent (as output parser)        | AI Agent                                 |                                                                                                |
| Send review request and wait for response | gotoHuman Node                  | Sends markdown for human review and waits | AI Agent                         | Approved?                                 |                                                                                                |
| Approved?                        | If Node                          | Checks if content was approved          | Send review request and wait for response | Convert to File                       |                                                                                                |
| Convert to File                  | Convert to File                   | Converts markdown string to file        | Approved?                         | Set file type as markdown                 |                                                                                                |
| Set file type as markdown         | Code Node (JavaScript)            | Sets MIME type to text/markdown         | Convert to File                   | Update file                              |                                                                                                |
| Update file                     | Google Drive                     | Updates Google Docs file with markdown  | Set file type as markdown          | None                                     |                                                                                                |
| Sticky Note1                    | Sticky Note                      | Visual note with gotoHuman logo         | None                             | None                                     | ![Source example](https://cdn1.gotohuman.com/public%2Fn8n-templates%2FgotoHuman%20full%20logo%201224px.png?alt=media) |
| Sticky Note                     | Sticky Note                      | Explains overall workflow steps         | None                             | None                                     | ## How It Works\n1. The AI Agent has access to other documents that provide enough context to write the content strategy. We ask it to generate the text in markdown format.\n2. To ensure our strategy document is not changed without our approval, we request a human review using **gotoHuman**. There the markdown content can be edited and properly previewed.\n3. Our workflow resumes once the review is completed. We check if the content was approved and then write the (potentially edited) markdown to our Google Docs file via the Google Drive node. |
| Sticky Note2                    | Sticky Note                      | Provides detailed usage and setup guide | None                             | None                                     | ## 🎯 Content Strategy Agent\n\nCollaborate with an AI Agent on a joint document, e.g. for creating your content marketing strategy, a sales plan, project status updates, or market analysis.\nThe AI Agent generates markdown text that you can review and edit it in **gotoHuman**, and only then is the existing Google Doc updated.  \nIn this example we use AI to update our company's content strategy for the next quarter.\n\n## How to set up\n1. **Most importantly, install the verified gotoHuman node before importing this template!**\n(Just add the node to a blank canvas before importing. Works with n8n cloud and self-hosted)\n2. Set up your credentials for gotoHuman, OpenAI, and Google Docs/Drive\n3. In gotoHuman, select and create the pre-built review template \"Strategy agent\" or import the ID: `F4sbcPEpyhNKBKbG9C1d`\n4. Select this template in the gotoHuman node\n\n## Requirements\nYou need accounts for\n- gotoHuman (human supervision)\n- OpenAI (Doc writing)\n- Google Docs/Drive\n\n## How to customize\n- Let the workflow run on a schedule, or create and connect a [manual trigger in gotoHuman](https://docs.gotohuman.com/manual-triggers) that lets you capture additional human input to feed your agent\n- Provide the agent with more context to write the content strategy\n- Use the gotoHuman response (or a Google Drive file change trigger) to run additional AI agents that can execute on the new strategy |
| Sticky Note3                    | Sticky Note                      | Shows screenshot of review interface    | None                             | None                                     | ![review](https://cdn1.gotohuman.com/public%2Fn8n-templates%2Fcontent-strategy-doc%2Fmd-review.JPG?alt=media) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node:**  
   - Name: `When clicking ‘Execute workflow’`  
   - Type: Manual Trigger (n8n-nodes-base.manualTrigger)  
   - No parameters needed.

2. **Create Google Docs Tool Nodes for Context:**  
   - Name: `Get company description`  
   - Type: Google Docs Tool (n8n-nodes-base.googleDocsTool)  
   - Operation: Get document content  
   - Configure credentials for Google Docs.  
   - Set Document ID or URL to your company description doc.

   - Name: `Get company strategy`  
   - Type: Google Docs Tool  
   - Operation: Get  
   - Configure with credentials and target Google Docs file for existing strategy.

3. **Create OpenAI Chat Model Node:**  
   - Name: `OpenAI Chat Model`  
   - Type: LangChain OpenAI Chat Model (@n8n/n8n-nodes-langchain.lmChatOpenAi)  
   - Model: Select GPT-4.1 from the list  
   - Configure OpenAI credentials with API key.

4. **Create Structured Output Parser Node:**  
   - Name: `Structured Output Parser`  
   - Type: LangChain Output Parser Structured (@n8n/n8n-nodes-langchain.outputParserStructured)  
   - Configure JSON schema example:  
     ```json
     {
       "strategyMarkdown": ""
     }
     ```

5. **Create AI Agent Node:**  
   - Name: `AI Agent`  
   - Type: LangChain AI Agent (@n8n/n8n-nodes-langchain.agent)  
   - Prompt: "Write a content strategy based on the company description and the given overall strategy. Consider the channels LinkedIn, Instagram and Email Newsletter"  
   - Add tools: `Get company description` and `Get company strategy`  
   - Assign language model: `OpenAI Chat Model`  
   - Assign output parser: `Structured Output Parser`  
   - Connect manual trigger (`When clicking ‘Execute workflow’`) to this node.

6. **Create gotoHuman Node:**  
   - Name: `Send review request and wait for response`  
   - Type: gotoHuman (@gotohuman/n8n-nodes-gotohuman.gotoHuman)  
   - Configure credentials for gotoHuman.  
   - Set review template ID to "Strategy agent" template or import ID `F4sbcPEpyhNKBKbG9C1d`  
   - Map field: `strategyMarkdown` set to `={{ $json.output.strategyMarkdown }}` from AI Agent output.  
   - Set to wait for webhook response.  
   - Connect AI Agent output to this node.

7. **Create If Node to Check Approval:**  
   - Name: `Approved?`  
   - Type: If Node (n8n-nodes-base.if)  
   - Condition: `$json.response === "approved"` (case sensitive)  
   - Connect gotoHuman node output to this node.

8. **Create Convert to File Node:**  
   - Name: `Convert to File`  
   - Type: Convert to File (n8n-nodes-base.convertToFile)  
   - Operation: toText  
   - Source Property: `responseValues.strategyMarkdown.value` (from gotoHuman review response)  
   - Connect `Approved?` node's "true" output to this node.

9. **Create Code Node to Set MIME Type:**  
   - Name: `Set file type as markdown`  
   - Type: Code (JavaScript) (n8n-nodes-base.code)  
   - Code:  
     ```javascript
     const item = $input.item;

     if (item.binary && item.binary.data) {
       item.binary.data.mimeType = 'text/markdown';
     }

     return item;
     ```  
   - Connect `Convert to File` output here.

10. **Create Google Drive Node to Update File:**  
    - Name: `Update file`  
    - Type: Google Drive (n8n-nodes-base.googleDrive)  
    - Operation: Update  
    - File ID: Specify the Google Docs file ID or URL to update  
    - Change File Content: true  
    - Connect `Set file type as markdown` to this node.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                | Context or Link                                                                                      |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| ## 🎯 Content Strategy Agent<br><br>Collaborate with an AI Agent on a joint document, e.g. for creating your content marketing strategy, a sales plan, project status updates, or market analysis. The AI Agent generates markdown text that you can review and edit it in **gotoHuman**, and only then is the existing Google Doc updated.<br>In this example we use AI to update our company's content strategy for the next quarter.<br><br>## How to set up<br>1. **Most importantly, install the verified gotoHuman node before importing this template!**<br>(Just add the node to a blank canvas before importing. Works with n8n cloud and self-hosted)<br>2. Set up your credentials for gotoHuman, OpenAI, and Google Docs/Drive<br>3. In gotoHuman, select and create the pre-built review template "Strategy agent" or import the ID: `F4sbcPEpyhNKBKbG9C1d`<br>4. Select this template in the gotoHuman node<br><br>## Requirements<br>You need accounts for<br>- gotoHuman (human supervision)<br>- OpenAI (Doc writing)<br>- Google Docs/Drive<br><br>## How to customize<br>- Let the workflow run on a schedule, or create and connect a [manual trigger in gotoHuman](https://docs.gotohuman.com/manual-triggers) that lets you capture additional human input to feed your agent<br>- Provide the agent with more context to write the content strategy<br>- Use the gotoHuman response (or a Google Drive file change trigger) to run additional AI agents that can execute on the new strategy | Sticky Note 2 content; for detailed setup and customization instructions.                         |
| ## How It Works<br>1. The AI Agent has access to other documents that provide enough context to write the content strategy. We ask it to generate the text in markdown format.<br>2. To ensure our strategy document is not changed without our approval, we request a human review using **gotoHuman**. There the markdown content can be edited and properly previewed.<br>3. Our workflow resumes once the review is completed. We check if the content was approved and then write the (potentially edited) markdown to our Google Docs file via the Google Drive node.                                                                                                                | Sticky Note overview explaining the workflow logic and process.                                  |
| ![Source example](https://cdn1.gotohuman.com/public%2Fn8n-templates%2FgotoHuman%20full%20logo%201224px.png?alt=media)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                | Sticky Note1, visual branding image related to gotoHuman.                                        |
| ![review](https://cdn1.gotohuman.com/public%2Fn8n-templates%2Fcontent-strategy-doc%2Fmd-review.JPG?alt=media)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           | Sticky Note3, screenshot illustrating the human review interface in gotoHuman.                   |

---

**Disclaimer:** The text provided is exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.