Basic Automatic Gmail Email Labelling with OpenAI and Gmail API

https://n8nworkflows.xyz/workflows/basic-automatic-gmail-email-labelling-with-openai-and-gmail-api-2740


# Basic Automatic Gmail Email Labelling with OpenAI and Gmail API

### 1. Workflow Overview

This workflow automates the categorization of incoming Gmail emails by leveraging the Gmail API and OpenAI’s language model. It periodically polls Gmail for new emails, analyzes their content, and applies the most relevant existing label. If no suitable label exists, it dynamically creates a new label following Gmail’s organizational conventions and assigns it to the email. The workflow also manages inbox cleanliness by removing less important emails (e.g., ads) from the inbox.

The workflow is logically divided into the following blocks:

- **1.1 Email Polling and Triggering:** Periodically checks Gmail for new incoming emails.
- **1.2 Email Content Retrieval:** Fetches the full content of each new email.
- **1.3 Label Management:** Reads existing Gmail labels and creates new labels if necessary.
- **1.4 AI-Powered Categorization Agent:** Uses OpenAI to analyze email content and decide on label assignment or creation.
- **1.5 Email Labeling and Inbox Management:** Applies labels to emails and manages inbox status.
- **1.6 Context Memory and Processing Control:** Maintains context across iterations and controls processing timing.

---

### 2. Block-by-Block Analysis

#### 2.1 Email Polling and Triggering

- **Overview:**  
  This block initiates the workflow by polling Gmail every 5 minutes to detect new incoming emails.

- **Nodes Involved:**  
  - Gmail Trigger  
  - Wait

- **Node Details:**

  - **Gmail Trigger**  
    - Type: Trigger node for Gmail API  
    - Configuration: Polls Gmail every 5 minutes for new emails without additional filters.  
    - Inputs: None (trigger node)  
    - Outputs: Triggers the workflow when new emails arrive.  
    - Credentials: Uses configured Gmail OAuth2 credentials.  
    - Edge Cases: Potential OAuth token expiration or API rate limits; network timeouts.  
    - Notes: Sticky note explains this node’s role as Gmail polling trigger.

  - **Wait**  
    - Type: Wait node  
    - Configuration: Waits for 1 second before proceeding to the next node.  
    - Inputs: Triggered by Gmail Trigger node.  
    - Outputs: Passes control to the AI agent node.  
    - Edge Cases: Minimal risk; used to buffer processing.

#### 2.2 Email Content Retrieval

- **Overview:**  
  Retrieves the full content of the email identified by the trigger to provide detailed data for analysis.

- **Nodes Involved:**  
  - Gmail - Get Message

- **Node Details:**

  - **Gmail - Get Message**  
    - Type: Gmail API tool node  
    - Configuration: Retrieves a specific email message by its ID, which is dynamically passed from the AI agent context (`$fromAI('gmail_message_id')`).  
    - Inputs: Receives message ID from AI agent node.  
    - Outputs: Provides full email content including subject, sender, recipients, and body.  
    - Credentials: Uses Gmail OAuth2 credentials.  
    - Edge Cases: Message ID might be invalid or deleted; API errors or permission issues.

#### 2.3 Label Management

- **Overview:**  
  Reads all existing Gmail labels to determine if a matching label exists for the email, and creates a new label if necessary.

- **Nodes Involved:**  
  - Gmail - Read Labels  
  - Gmail - Create Label

- **Node Details:**

  - **Gmail - Read Labels**  
    - Type: Gmail API tool node  
    - Configuration: Fetches all existing Gmail labels, returning the full list for comparison.  
    - Inputs: Triggered by AI agent node.  
    - Outputs: List of labels available in the Gmail account.  
    - Credentials: Gmail OAuth2 credentials.  
    - Edge Cases: API rate limits or permission errors.

  - **Gmail - Create Label**  
    - Type: Gmail API tool node  
    - Configuration: Creates a new Gmail label if the AI agent determines no existing label matches. The label name is dynamically provided by the AI agent (`$fromAI('new_label_name')`).  
    - Inputs: Label name from AI agent.  
    - Outputs: Confirmation of label creation with label ID.  
    - Credentials: Gmail OAuth2 credentials.  
    - Edge Cases: Label name conflicts, API errors, or permission issues.

#### 2.4 AI-Powered Categorization Agent

- **Overview:**  
  This is the core logic block where OpenAI’s Chat model analyzes the email content, compares it with existing labels, and decides on label assignment or creation.

- **Nodes Involved:**  
  - Gmail labelling agent (LangChain Agent)  
  - OpenAI Chat Model1

- **Node Details:**

  - **Gmail labelling agent**  
    - Type: LangChain Agent node integrating AI with Gmail tools  
    - Configuration:  
      - Uses system prompt defining objectives and instructions for email categorization.  
      - Integrates tools: Get message, Read labels, Create label, Add label to message.  
      - Handles label matching, assignment, and creation logic.  
      - Removes inbox label for less important emails (ads/promotions).  
      - Retries up to 5 iterations per email to refine categorization.  
      - On error, continues workflow without stopping.  
    - Inputs: Receives email data from Wait node and AI responses from OpenAI Chat Model1.  
    - Outputs: Provides label IDs and new label names to Gmail tool nodes.  
    - Edge Cases: AI model misclassification, API failures, expression evaluation errors.  
    - Notes: Sticky note details the agent’s objectives, tools, and instructions.

  - **OpenAI Chat Model1**  
    - Type: OpenAI Chat model node  
    - Configuration: Uses OpenAI’s language model with max tokens set to 4096 for detailed analysis.  
    - Inputs: Receives prompt text from Gmail labelling agent.  
    - Outputs: Provides AI-generated responses for label suggestions and decisions.  
    - Credentials: OpenAI API key configured.  
    - Edge Cases: API rate limits, token limits, network errors.

#### 2.5 Email Labeling and Inbox Management

- **Overview:**  
  Applies the chosen label(s) to the email and manages inbox status by removing the inbox label for less important emails.

- **Nodes Involved:**  
  - Gmail - Add Label to Message

- **Node Details:**

  - **Gmail - Add Label to Message**  
    - Type: Gmail API tool node  
    - Configuration: Adds one or more label IDs to the email message. Label IDs are dynamically passed from the AI agent (`$fromAI('gmail_categories')`).  
    - Inputs: Receives message ID and label IDs from AI agent.  
    - Outputs: Confirmation of label assignment.  
    - Credentials: Gmail OAuth2 credentials.  
    - Edge Cases: Label IDs invalid, message ID invalid, API errors.

#### 2.6 Context Memory and Processing Control

- **Overview:**  
  Maintains conversational context for the AI agent across multiple emails and controls processing flow.

- **Nodes Involved:**  
  - Window Buffer Memory

- **Node Details:**

  - **Window Buffer Memory**  
    - Type: LangChain Memory Buffer node  
    - Configuration: Uses the email ID as a session key to maintain context per email.  
    - Inputs: Connected to AI agent node to provide memory context.  
    - Outputs: Supplies memory context for AI analysis.  
    - Edge Cases: Memory overflow or session key conflicts.

---

### 3. Summary Table

| Node Name               | Node Type                          | Functional Role                          | Input Node(s)          | Output Node(s)               | Sticky Note                                                                                                  |
|-------------------------|----------------------------------|----------------------------------------|------------------------|-----------------------------|--------------------------------------------------------------------------------------------------------------|
| Gmail Trigger           | n8n-nodes-base.gmailTrigger       | Polls Gmail every 5 minutes for emails | None                   | Wait                        | ## Gmail trigger<br>Poll Gmail every x minutes, trigger when a new email is received.<br>- Gmail API          |
| Wait                    | n8n-nodes-base.wait               | Adds a short delay before processing   | Gmail Trigger          | Gmail labelling agent        |                                                                                                              |
| Gmail labelling agent   | @n8n/n8n-nodes-langchain.agent   | AI agent for email categorization      | Wait, OpenAI Chat Model1, Gmail - Get Message, Gmail - Read Labels, Gmail - Create Label, Gmail - Add Label to Message, Window Buffer Memory | Gmail - Get Message, Gmail - Read Labels, Gmail - Create Label, Gmail - Add Label to Message | ## Gmail labelling agent<br>- Read the message<br>- Read existing labels<br>- Create a new label if needed<br>- Assign label to message<br><br>Objective:<br>Automatically categorize incoming emails based on existing Gmail labels or create a new label if none match.<br><br>Tools:<br>- Get message<br>- Read all labels<br>- Create label<br>- Assign label to message<br><br>Instructions:<br>Label Matching:<br>Analyze the email's subject, sender, recipient, keywords, and content.<br>Compare with existing Gmail labels to find the most relevant match.<br>Label Assignment:<br>Assign the email to the most appropriate existing label.`<br>Remove the inbox label if the email is of less importance (like ads, promotions, aka "Reclame"), keep normal and important emails in the inbox.<br>If no suitable label exists, create a new label based on the existing labels. Try reusing existing labels as much as possible. Always create a label as a sublabel, if no label applies, if the main label already exists, create the new label under the existing label, if no main label exists, create the label AI and create the new label under this label.<br>Label Creation:<br>Ensure new labels align with the structure of existing ones, including capitalization, delimiters, and prefixes.<br>Examples:<br>If the email subject is "Project Alpha Update," assign to [Project Alpha] if it exists.<br>For "New Vendor Inquiry," create "Vendor Inquiry" if no relevant label exists.<br>Outcome:<br>Emails are consistently categorized under the appropriate or newly created labels, maintaining Gmail's organizational structure. |
| OpenAI Chat Model1      | @n8n/n8n-nodes-langchain.lmChatOpenAi | Provides AI analysis for categorization | Gmail labelling agent   | Gmail labelling agent        | ## OpenAI<br>- Add credentials                                                                                 |
| Gmail - Get Message     | n8n-nodes-base.gmailTool          | Retrieves full email content           | Gmail labelling agent   | Gmail labelling agent        |                                                                                                              |
| Gmail - Read Labels     | n8n-nodes-base.gmailTool          | Reads all existing Gmail labels        | Gmail labelling agent   | Gmail labelling agent        |                                                                                                              |
| Gmail - Create Label    | n8n-nodes-base.gmailTool          | Creates a new Gmail label if needed    | Gmail labelling agent   | Gmail labelling agent        |                                                                                                              |
| Gmail - Add Label to Message | n8n-nodes-base.gmailTool      | Adds label(s) to the email message     | Gmail labelling agent   | None                        |                                                                                                              |
| Window Buffer Memory    | @n8n/n8n-nodes-langchain.memoryBufferWindow | Maintains AI context per email          | Gmail labelling agent   | Gmail labelling agent        |                                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Gmail Trigger Node**  
   - Type: Gmail Trigger  
   - Configure to poll Gmail every 5 minutes (set mode to "everyX", unit "minutes", value 5)  
   - Add Gmail OAuth2 credentials  
   - Position: Start of workflow  

2. **Create Wait Node**  
   - Type: Wait  
   - Set wait time to 1 second  
   - Connect Gmail Trigger output to Wait input  

3. **Create Gmail Labelling Agent Node (LangChain Agent)**  
   - Type: LangChain Agent  
   - Configure system prompt with detailed instructions for email categorization, label matching, creation, and assignment (use the provided objective and instructions)  
   - Set max iterations to 5  
   - Set onError to continue workflow on failure  
   - Connect Wait node output to this node input  

4. **Create OpenAI Chat Model Node**  
   - Type: OpenAI Chat Model (LangChain)  
   - Set max tokens to 4096  
   - Add OpenAI API credentials  
   - Connect this node as the AI language model input for the Gmail labelling agent node  

5. **Create Gmail - Get Message Node**  
   - Type: Gmail Tool  
   - Operation: Get message by ID  
   - Set messageId parameter to: `={{ $fromAI('gmail_message_id', 'id of the gmail message, like 1944fdc33f544369', 'string') }}`  
   - Add Gmail OAuth2 credentials  
   - Connect this node as an AI tool input to the Gmail labelling agent node  

6. **Create Gmail - Read Labels Node**  
   - Type: Gmail Tool  
   - Resource: Label  
   - Operation: Return all labels (returnAll = true)  
   - Add Gmail OAuth2 credentials  
   - Connect this node as an AI tool input to the Gmail labelling agent node  

7. **Create Gmail - Create Label Node**  
   - Type: Gmail Tool  
   - Resource: Label  
   - Operation: Create  
   - Set name parameter to: `={{ $fromAI('new_label_name', 'new label name', 'string') }}`  
   - Add Gmail OAuth2 credentials  
   - Connect this node as an AI tool input to the Gmail labelling agent node  

8. **Create Gmail - Add Label to Message Node**  
   - Type: Gmail Tool  
   - Operation: Add labels  
   - Set labelIds parameter to: `={{ $fromAI('gmail_categories', 'array of label ids') }}`  
   - Set messageId parameter to: `={{ $fromAI('gmail_message_id') }}`  
   - Add Gmail OAuth2 credentials  
   - Connect this node as an AI tool input to the Gmail labelling agent node  

9. **Create Window Buffer Memory Node**  
   - Type: LangChain Memory Buffer Window  
   - Set sessionKey to: `={{ $json.id }}` (email ID)  
   - Set sessionIdType to "customKey"  
   - Connect this node as AI memory input to the Gmail labelling agent node  

10. **Add Sticky Notes (Optional but Recommended)**  
    - Add notes near Gmail Trigger explaining polling purpose.  
    - Add notes near Gmail labelling agent explaining objectives, tools, and instructions.  
    - Add notes near OpenAI Chat Model and Gmail nodes reminding to configure credentials.  

11. **Activate the Workflow**  
    - Ensure all credentials are properly configured (Gmail OAuth2 and OpenAI API key).  
    - Activate workflow to start polling and processing emails automatically every 5 minutes.

---

### 5. General Notes & Resources

| Note Content                                                                                                                     | Context or Link                                                                                  |
|----------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| You can improve the AI prompt by adding more personal rules on how to categorize emails for better results.                      | Workflow description and AI agent instructions                                                 |
| Gmail API OAuth2 credentials must be configured in n8n for all Gmail nodes to function correctly.                                | Gmail API setup prerequisite                                                                    |
| OpenAI API key must be configured in n8n for the OpenAI Chat Model node to analyze email content.                                | OpenAI API setup prerequisite                                                                   |
| The workflow is ideal for users with high email volume who want automated, consistent inbox organization.                       | Use case description                                                                            |
| Label creation logic ensures new labels are sublabels under existing main labels or under a default "AI" label if none exists.  | Gmail labelling agent instructions                                                             |
| Removing inbox label for less important emails (ads, promotions) helps maintain inbox cleanliness.                              | Gmail labelling agent instructions                                                             |

---

This documentation provides a complete, detailed reference for understanding, reproducing, and modifying the "Basic Automatic Gmail Email Labelling with OpenAI and Gmail API" workflow. It covers all nodes, their configurations, interconnections, and operational logic to ensure robust implementation and maintenance.